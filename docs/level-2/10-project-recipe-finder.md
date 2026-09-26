---
description: "Level 2 project: a routed Recipe Finder SPA with debounced search, fetch hooks, URL state, favourites in context with a reducer, and Vitest tests."
---

# 10 · Project — Recipe Finder SPA

This project ties Level 2 together: routing, data fetching with proper loading and error
states, shared state through context and a reducer, custom hooks, a sensible amount of
memoization, and tests.

## What you'll build

- `/` — search recipes by name; the query lives in the URL (`/?q=paneer`).
- `/recipe/:id` — recipe details: image, ingredients, instructions.
- `/favourites` — recipes you've starred, persisted in `localStorage`.
- A header with a favourites count visible on every page.

Data comes from **TheMealDB**, a free recipe API. Its public test key `1` is embedded in
the URL and is intended for development and education; check their site's terms before
using it in anything public. Endpoints used:

- `https://www.themealdb.com/api/json/v1/1/search.php?s=<query>`
- `https://www.themealdb.com/api/json/v1/1/lookup.php?i=<id>`

Both return `{ meals: [...] }`, or `{ meals: null }` when nothing matches.

## Structure

```text
src/
├── main.jsx
├── App.jsx                 # routes
├── api.js                  # URL builders + response normalisation
├── hooks/
│   ├── useFetch.js         # from lesson 6
│   └── useDebouncedValue.js# from lesson 5
├── favourites/
│   ├── favourites-context.jsx
│   └── favouritesReducer.js
├── components/
│   ├── Layout.jsx
│   ├── RecipeCard.jsx
│   └── FavouriteButton.jsx
└── pages/
    ├── SearchPage.jsx
    ├── RecipePage.jsx
    ├── FavouritesPage.jsx
    └── NotFound.jsx
```

```bash
npm create vite@latest recipe-finder -- --template react
cd recipe-finder
npm install react-router
npm install -D vitest jsdom @testing-library/react @testing-library/dom @testing-library/user-event @testing-library/jest-dom
```

## Step 1 — the API module

Keep API shapes out of components. Normalise once:

```js
// src/api.js
const BASE = 'https://www.themealdb.com/api/json/v1/1'

export const searchUrl = q => `${BASE}/search.php?s=${encodeURIComponent(q)}`
export const recipeUrl = id => `${BASE}/lookup.php?i=${encodeURIComponent(id)}`

export function toRecipe(meal) {
  const ingredients = []
  for (let i = 1; i <= 20; i++) {
    const name = meal[`strIngredient${i}`]?.trim()
    const measure = meal[`strMeasure${i}`]?.trim()
    if (name) ingredients.push({ name, measure })
  }
  return {
    id: meal.idMeal,
    name: meal.strMeal,
    image: meal.strMealThumb,
    category: meal.strCategory,
    area: meal.strArea,
    instructions: meal.strInstructions,
    ingredients,
  }
}
```

TheMealDB stores ingredients as `strIngredient1` … `strIngredient20`; normalising here
means the rest of the app sees a clean `ingredients` array.

## Step 2 — favourites: reducer + context

```js
// src/favourites/favouritesReducer.js
export function favouritesReducer(state, action) {
  switch (action.type) {
    case 'toggled': {
      const exists = state.some(r => r.id === action.recipe.id)
      return exists
        ? state.filter(r => r.id !== action.recipe.id)
        : [...state, { id: action.recipe.id, name: action.recipe.name, image: action.recipe.image }]
    }
    case 'cleared':
      return []
    default:
      throw new Error(`Unknown action ${action.type}`)
  }
}
```

```jsx
// src/favourites/favourites-context.jsx
import { createContext, useContext, useEffect, useReducer } from 'react'
import { favouritesReducer } from './favouritesReducer'

const KEY = 'recipe-finder:favourites'
const FavouritesContext = createContext(null)
const FavouritesDispatchContext = createContext(null)

function load() {
  try {
    const v = JSON.parse(localStorage.getItem(KEY))
    return Array.isArray(v) ? v : []
  } catch {
    return []
  }
}

export function FavouritesProvider({ children }) {
  const [favourites, dispatch] = useReducer(favouritesReducer, undefined, load)

  useEffect(() => {
    localStorage.setItem(KEY, JSON.stringify(favourites))
  }, [favourites])

  return (
    <FavouritesContext.Provider value={favourites}>
      <FavouritesDispatchContext.Provider value={dispatch}>{children}</FavouritesDispatchContext.Provider>
    </FavouritesContext.Provider>
  )
}

export function useFavourites() {
  const ctx = useContext(FavouritesContext)
  if (ctx === null) throw new Error('useFavourites must be used inside <FavouritesProvider>')
  return ctx
}

export function useFavouritesDispatch() {
  const ctx = useContext(FavouritesDispatchContext)
  if (ctx === null) throw new Error('useFavouritesDispatch must be used inside <FavouritesProvider>')
  return ctx
}
```

Only a small summary (`id`, `name`, `image`) is stored, not the full recipe — the detail
page fetches the rest. `useReducer(reducer, undefined, load)` reads storage only once.

## Step 3 — shared components

```jsx
// src/components/FavouriteButton.jsx
import { useFavourites, useFavouritesDispatch } from '../favourites/favourites-context'

export default function FavouriteButton({ recipe }) {
  const favourites = useFavourites()
  const dispatch = useFavouritesDispatch()
  const isFav = favourites.some(r => r.id === recipe.id)
  return (
    <button
      type="button"
      aria-pressed={isFav}
      aria-label={isFav ? `Remove ${recipe.name} from favourites` : `Add ${recipe.name} to favourites`}
      onClick={() => dispatch({ type: 'toggled', recipe })}
    >
      {isFav ? '★' : '☆'}
    </button>
  )
}
```

```jsx
// src/components/RecipeCard.jsx
import { Link } from 'react-router'
import FavouriteButton from './FavouriteButton'

export default function RecipeCard({ recipe }) {
  return (
    <article className="recipe-card">
      <img src={`${recipe.image}/small`} alt="" width={120} height={120} loading="lazy" />
      <div>
        <h3><Link to={`/recipe/${recipe.id}`}>{recipe.name}</Link></h3>
        {recipe.area && <p>{recipe.area} · {recipe.category}</p>}
      </div>
      <FavouriteButton recipe={recipe} />
    </article>
  )
}
```

TheMealDB serves smaller thumbnails when `/small` is appended to the image URL; if that
ever changes, drop the suffix. `alt=""` marks the image decorative because the link text
already names the recipe.

```jsx
// src/components/Layout.jsx
import { NavLink, Outlet } from 'react-router'
import { useFavourites } from '../favourites/favourites-context'

export default function Layout() {
  const favourites = useFavourites()
  return (
    <div className="app">
      <header>
        <NavLink to="/" end>Search</NavLink>
        <NavLink to="/favourites">Favourites ({favourites.length})</NavLink>
      </header>
      <main><Outlet /></main>
    </div>
  )
}
```

## Step 4 — pages

```jsx
// src/pages/SearchPage.jsx
import { useMemo } from 'react'
import { useSearchParams } from 'react-router'
import { useDebouncedValue } from '../hooks/useDebouncedValue'
import { useFetch } from '../hooks/useFetch'
import { searchUrl, toRecipe } from '../api'
import RecipeCard from '../components/RecipeCard'

export default function SearchPage() {
  const [params, setParams] = useSearchParams()
  const q = params.get('q') ?? ''
  const debounced = useDebouncedValue(q.trim(), 400)
  const { status, data, error, reload } = useFetch(debounced ? searchUrl(debounced) : null)

  // Normalising 20 ingredient fields per result isn't free; only redo it when data changes.
  const recipes = useMemo(() => (data?.meals ?? []).map(toRecipe), [data])

  return (
    <section>
      <label htmlFor="search">Search recipes</label>
      <input
        id="search"
        type="search"
        value={q}
        onChange={e => setParams(e.target.value ? { q: e.target.value } : {}, { replace: true })}
        placeholder="e.g. biryani, pasta, soup"
      />

      {!debounced && <p>Start typing to find recipes.</p>}
      {debounced && status === 'loading' && <p aria-busy="true">Searching…</p>}
      {debounced && status === 'error' && (
        <p role="alert">Search failed ({error.message}). <button onClick={reload}>Retry</button></p>
      )}
      {debounced && status === 'success' && recipes.length === 0 && <p>No recipes match “{debounced}”.</p>}
      {debounced && status === 'success' && recipes.length > 0 && (
        <div className="grid">
          {recipes.map(r => <RecipeCard key={r.id} recipe={r} />)}
        </div>
      )}
    </section>
  )
}
```

`replace: true` on each keystroke avoids filling history with one entry per character.

Note the `useMemo` here earns its place for a *reference-stability* reason too: without
it, `recipes` would be a new array every render.

```jsx
// src/pages/RecipePage.jsx
import { useParams, Link } from 'react-router'
import { useFetch } from '../hooks/useFetch'
import { recipeUrl, toRecipe } from '../api'
import FavouriteButton from '../components/FavouriteButton'

export default function RecipePage() {
  const { id } = useParams()
  const { status, data, error, reload } = useFetch(recipeUrl(id))

  if (status === 'loading') return <p aria-busy="true">Loading recipe…</p>
  if (status === 'error') return <p role="alert">{error.message} <button onClick={reload}>Retry</button></p>
  if (!data?.meals) return <p>Recipe not found. <Link to="/">Back to search</Link></p>

  const recipe = toRecipe(data.meals[0])
  return (
    <article>
      <h1>{recipe.name} <FavouriteButton recipe={recipe} /></h1>
      <img src={recipe.image} alt={recipe.name} width={360} />
      <h2>Ingredients</h2>
      <ul>
        {recipe.ingredients.map((ing, i) => (
          <li key={`${ing.name}-${i}`}>{ing.measure} {ing.name}</li>
        ))}
      </ul>
      <h2>Method</h2>
      {recipe.instructions.split(/\r?\n/).filter(Boolean).map((para, i) => <p key={i}>{para}</p>)}
    </article>
  )
}
```

Index keys are fine for the instruction paragraphs: the list is static for this render,
never reordered, and holds no state.

```jsx
// src/pages/FavouritesPage.jsx
import { useFavourites, useFavouritesDispatch } from '../favourites/favourites-context'
import RecipeCard from '../components/RecipeCard'

export default function FavouritesPage() {
  const favourites = useFavourites()
  const dispatch = useFavouritesDispatch()
  if (favourites.length === 0) return <p>No favourites yet — star a recipe to save it here.</p>
  return (
    <section>
      <button onClick={() => dispatch({ type: 'cleared' })}>Clear all</button>
      <div className="grid">{favourites.map(r => <RecipeCard key={r.id} recipe={r} />)}</div>
    </section>
  )
}
```

## Step 5 — routes

```jsx
// src/App.jsx
import { Routes, Route } from 'react-router'
import Layout from './components/Layout'
import SearchPage from './pages/SearchPage'
import RecipePage from './pages/RecipePage'
import FavouritesPage from './pages/FavouritesPage'
import NotFound from './pages/NotFound'

export default function App() {
  return (
    <Routes>
      <Route element={<Layout />}>
        <Route index element={<SearchPage />} />
        <Route path="recipe/:id" element={<RecipePage />} />
        <Route path="favourites" element={<FavouritesPage />} />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  )
}
```

```jsx
// src/main.jsx
createRoot(document.getElementById('root')).render(
  <StrictMode>
    <BrowserRouter>
      <FavouritesProvider>
        <App />
      </FavouritesProvider>
    </BrowserRouter>
  </StrictMode>,
)
```

`NotFound` is a one-liner with a `Link` home — write it yourself.

## Step 6 — tests

```js
// src/favourites/favouritesReducer.test.js
import { favouritesReducer } from './favouritesReducer'

const r = { id: '1', name: 'Dal', image: 'x' }

test('toggle adds then removes', () => {
  const added = favouritesReducer([], { type: 'toggled', recipe: r })
  expect(added).toEqual([r])
  expect(favouritesReducer(added, { type: 'toggled', recipe: r })).toEqual([])
})
```

```jsx
// src/pages/SearchPage.test.jsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MemoryRouter } from 'react-router'
import { vi, beforeEach, afterEach } from 'vitest'
import SearchPage from './SearchPage'
import { FavouritesProvider } from '../favourites/favourites-context'

beforeEach(() => localStorage.clear())
afterEach(() => vi.unstubAllGlobals())

function renderPage(url = '/') {
  return render(
    <MemoryRouter initialEntries={[url]}>
      <FavouritesProvider><SearchPage /></FavouritesProvider>
    </MemoryRouter>,
  )
}

test('shows results for the query in the URL', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: true,
    json: async () => ({ meals: [{ idMeal: '52772', strMeal: 'Teriyaki Chicken', strMealThumb: 'img' }] }),
  }))
  renderPage('/?q=chicken')
  expect(await screen.findByRole('link', { name: 'Teriyaki Chicken' })).toBeInTheDocument()
})

test('can favourite a result', async () => {
  const user = userEvent.setup()
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: true,
    json: async () => ({ meals: [{ idMeal: '1', strMeal: 'Dal', strMealThumb: 'img' }] }),
  }))
  renderPage('/?q=dal')
  await user.click(await screen.findByRole('button', { name: /add dal to favourites/i }))
  expect(screen.getByRole('button', { name: /remove dal from favourites/i })).toHaveAttribute('aria-pressed', 'true')
})
```

The mocked meal objects omit most fields; `toRecipe` handles missing ingredient fields via
optional chaining, which the test implicitly verifies.

## How It Actually Works

Follow a search end to end. Typing "dal" calls `setParams` three times; each updates the
URL with `history.replaceState` and the router's location state, re-rendering
`SearchPage`. `q` changes immediately (the input stays responsive), but `debounced`
only changes 400 ms after the last keystroke because each keystroke's effect cleanup
cancels the previous timer. When `debounced` changes, `useFetch` receives a new URL,
its effect aborts any in-flight request, and a new request starts.

When the response arrives, one `setState` flips status to `'success'`. `useMemo` sees a
new `data` reference and normalises the results; `RecipeCard`s are reconciled by recipe
id. Starring a recipe dispatches to the favourites reducer; the favourites context value
(a new array) changes, so `Layout`'s counter and every `FavouriteButton` re-render — but
`SearchPage` itself doesn't re-render (it doesn't read favourites), so no refetch and no
re-normalisation happen. That's the dependency tracking of context working for you.

Refreshing `/recipe/52772` works in dev because Vite's dev server falls back to
`index.html`; after deployment you need the host rewrite from lesson 7.

## Exercise — extend it

1. Add a "Browse by category" page using
   `https://www.themealdb.com/api/json/v1/1/categories.php` and
   `filter.php?c=<category>`, reusing `RecipeCard`.
2. Add a "Shopping list" that combines ingredients from selected favourites into one
   de-duplicated list.
3. Cache detail responses in memory (keyed by id) so returning to a recipe is instant —
   and write down what would go wrong if the data could change on the server.
4. Add tests for the error state and the "no results" state of `SearchPage`.
