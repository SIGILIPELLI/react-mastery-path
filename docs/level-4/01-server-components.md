---
description: "React Server Components explained: the server/client boundary, 'use client', serializable props, Server Functions, streaming, and framework support."
---

# 01 · Server Components & Frameworks

Everything so far ran in the browser: React downloaded, your components downloaded,
then data was fetched from an API. **React Server Components (RSC)** add a second kind
of component that runs **only on the server** (or at build time). Their code never
ships to the browser; only their rendered output does.

RSC is a React feature, but using it requires a framework or bundler integration that
implements the server side — Next.js's App Router is the most widely used; React Router's
framework mode and others have been adding support. You can't enable Server Components
in a plain Vite client app by flipping a switch. This lesson teaches the
framework-agnostic model; examples use conventions shared by RSC frameworks, and you
should confirm file layout and config in your framework's docs.

## Two kinds of components

| | Server Components | Client Components |
|---|---|---|
| Where they run | server / build time only | server (for the initial HTML) **and** browser |
| Can use state, effects, event handlers | no | yes |
| Can use browser APIs | no | yes (in effects/handlers) |
| Can read files, databases, secrets directly | yes | no |
| Code sent to browser | none | yes |
| Can be `async` | yes | no |

In an RSC app, components are **Server Components by default**. You opt a module into the
client with the `'use client'` directive at the top of the file.

## A Server Component

```jsx
// app/products/page.jsx  (a Server Component — no directive)
import { db } from '@/lib/db'          // server-only module: DB client
import AddToCartButton from './AddToCartButton'

export default async function ProductsPage() {
  const products = await db.product.findMany({ orderBy: { name: 'asc' } })
  return (
    <ul>
      {products.map(p => (
        <li key={p.id}>
          {p.name} — ₹{p.price}
          <AddToCartButton productId={p.id} />
        </li>
      ))}
    </ul>
  )
}
```

`db` here stands for whatever data layer you use (an ORM, SQL client or internal API).
No `useEffect`, no loading state, no API endpoint written just for this page: the
component awaits its data where the data lives. The database library and query code add
zero bytes to the browser bundle.

## A Client Component

```jsx
// app/products/AddToCartButton.jsx
'use client'

import { useState } from 'react'

export default function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false)
  return (
    <button onClick={() => setAdded(true)} disabled={added}>
      {added ? 'Added ✓' : 'Add to cart'}
    </button>
  )
}
```

`'use client'` marks a **boundary**: this module and everything it imports become part
of the client bundle. It doesn't mean "render only in the browser" — Client Components
are still pre-rendered to HTML on the server for the first load, then **hydrated**.

## The rules of the boundary

1. **Props from Server to Client Components must be serializable**: strings, numbers,
   booleans, `null`, plain objects and arrays, `Date`, `Map`/`Set`, promises, JSX, and
   Server Functions. **Not** ordinary functions or class instances.

   ```jsx
   // ❌ can't pass a function across the boundary
   <AddToCartButton onAdd={() => console.log('hi')} />
   ```

2. **Client modules can't import Server Components** (the import would pull server code
   into the client bundle). But a Client Component can **render** Server Components
   passed to it as `children` or props:

   ```jsx
   // Server Component
   <ClientTabs>
     <ServerRenderedReport />   {/* rendered on the server, passed as a prop */}
   </ClientTabs>
   ```

3. **Keep `'use client'` low in the tree.** Mark the interactive leaf (the button), not
   the whole page, so most of the page remains server-only.

4. **Guard server-only code.** Importing the `server-only` package in a module makes the
   build fail if a client module ever imports it — protection against leaking secrets.

## Server Functions (`'use server'`)

A function marked `'use server'` runs on the server but can be called from the client
— the framework creates an endpoint for it automatically:

```jsx
// app/actions.js
'use server'

import { db } from '@/lib/db'
import { z } from 'zod'

const Schema = z.object({ productId: z.string(), qty: z.coerce.number().int().min(1).max(10) })

export async function addToCart(prevState, formData) {
  const parsed = Schema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: 'Invalid quantity' }
  // authenticate and authorise here — this is a public endpoint!
  await db.cartItem.create({ data: parsed.data })
  return { ok: true }
}
```

```jsx
'use client'
import { useActionState } from 'react'
import { addToCart } from '../actions'

export function AddToCartForm({ productId }) {
  const [state, action, pending] = useActionState(addToCart, {})
  return (
    <form action={action}>
      <input type="hidden" name="productId" value={productId} />
      <input name="qty" type="number" defaultValue={1} min={1} max={10} />
      <button disabled={pending}>Add</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  )
}
```

Because the form's `action` is a Server Function, frameworks can make it work **before
JavaScript loads** (progressive enhancement). Treat every Server Function like a public
API route: validate input and check authorisation inside it. Being "in a file next to
your component" doesn't make it private.

## Streaming with Suspense

Slow data doesn't have to block the page:

```jsx
export default function Dashboard() {
  return (
    <>
      <Header />                                   {/* sent immediately */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />                           {/* async; streams in when ready */}
      </Suspense>
    </>
  )
}
```

The server sends the shell with the fallback, keeps the connection open, and streams the
chart's output when its data resolves; a small inline script swaps it into place.

## Choosing a framework (or not)

- **Client-only SPA (Vite + React Router data mode + TanStack Query)**: great for apps
  behind a login where SEO doesn't matter, with a separate API. Simple hosting (static
  files).
- **A React framework with server rendering / RSC** (e.g. Next.js App Router, React
  Router framework mode): public content, SEO, fast first paint on slow devices, data
  close to the UI. Requires a server or serverless runtime, and brings more concepts
  (caching layers, server/client boundaries).
- **Static-site generation** (content known at build time): docs, blogs, marketing
  pages; most frameworks support it.

Pick by constraints — who the users are, where the data is, what hosting you have — not
by trend.

## How It Actually Works

When a request arrives, the framework renders the Server Component tree on the server.
Server Components are called (awaited if async) and their output is resolved down to host
elements. When rendering reaches a Client Component, React does **not** run it here for
the RSC payload; it emits a **reference** instead: "module `AddToCartButton.jsx#default`
with props `{ productId: 'p1' }`". The result is the **RSC payload**, a serialized
description of the tree (a streaming, line-based format) containing host elements, text,
client references and serialized props.

For the initial page load, the framework then runs a regular server-side render of that
tree — this time actually executing Client Components — to produce HTML, and sends both
the HTML and the RSC payload. In the browser, React loads the client modules referenced
in the payload and **hydrates**: it walks the existing HTML, attaches event handlers and
state to the Client Components, and doesn't re-create DOM that already matches.

On later navigations, the browser fetches **only the RSC payload** for the new route,
and React reconciles it into the existing tree like any other update — so client state
in layouts that didn't change is preserved.

The bundler integration is what makes the directives work: `'use client'` tells it to
treat the module as a client entry point (include it in client chunks and give it an ID
referenceable from the payload); `'use server'` tells it to replace the function in
client code with a stub that POSTs its serialized arguments to a generated endpoint,
which invokes the real function on the server.

## Common mistakes

- **`'use client'` at the top of every file** — you lose the benefits and ship
  everything.
- **Passing non-serializable props** (functions, class instances) from server to client.
- **Assuming Server Functions are private** — they're callable endpoints; validate and
  authorise.
- **Leaking secrets** by importing a module with an API key into a client module. Use
  `server-only` and environment variable naming conventions of your framework.
- **Using hooks or browser APIs in a Server Component** (`useState`, `window`) → build or
  runtime errors.

## Exercise

Using an RSC-capable framework of your choice (follow its official getting-started guide):

1. Build a `/products` page as a Server Component reading from a local JSON file with
   `fs/promises` (no API route).
2. Add a Client Component "favourite" toggle per product, keeping `'use client'` on the
   smallest component possible. Check in the browser's Network/Sources panels that the
   JSON-reading code is absent from the client bundle.
3. Add a Server Function that appends a review to the JSON file, called from a form with
   `useActionState`, validating input with Zod. Try submitting with JavaScript disabled.
4. Wrap a deliberately slow (`await new Promise(r => setTimeout(r, 2000))`) section in
   `Suspense` and watch the HTML stream in.
5. Write down which parts of your Level 3 Kanban app would benefit from being Server
   Components and which must stay client-side, and why.
