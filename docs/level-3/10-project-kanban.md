---
description: "Level 3 project: a typed React Kanban board with TanStack Query, optimistic moves, Zustand UI state, accessible drag and drop, and code-splitting."
---

# 10 · Project — Kanban Board

This project exercises the whole of Level 3 in one app: a Kanban board (think a
minimal Trello) with columns and cards, written in TypeScript. Cards live on a local
REST server and are managed with TanStack Query, including optimistic moves. UI-only
state (selected card, filters) lives in a small Zustand store. Cards can be moved by
drag and drop **and** by keyboard. The card editor is lazy-loaded inside a native
`<dialog>`, and failures are contained by error boundaries.

## Features

- Three columns: To do, In progress, Done.
- Create, edit and delete cards; move them between columns.
- Drag and drop with the mouse; move with keyboard buttons as an accessible alternative.
- Filter by text and by label.
- Optimistic updates with rollback when the server fails.
- A lazy-loaded card editor dialog.

## Setup

```bash
npm create vite@latest kanban -- --template react-ts
cd kanban
npm install @tanstack/react-query zustand react-error-boundary zod
npm install -D json-server
```

Create `db.json` at the project root:

```json
{
  "cards": [
    { "id": "1", "title": "Set up CI", "description": "", "column": "todo", "label": "ops", "order": 1 },
    { "id": "2", "title": "Design login screen", "description": "", "column": "doing", "label": "design", "order": 1 },
    { "id": "3", "title": "Write README", "description": "", "column": "done", "label": "docs", "order": 1 }
  ]
}
```

Add a script and run it in a second terminal:

```json
"api": "json-server --watch db.json --port 3001"
```

`json-server` gives you `GET/POST /cards` and `GET/PATCH/DELETE /cards/:id`, persisted to
`db.json`. Its CLI flags have changed across major versions; if `--watch` is rejected,
run `npx json-server db.json --port 3001` instead.

## Step 1 — types and validation

```ts
// src/types.ts
import { z } from 'zod'

export const COLUMNS = ['todo', 'doing', 'done'] as const
export type ColumnId = (typeof COLUMNS)[number]
export const COLUMN_TITLES: Record<ColumnId, string> = { todo: 'To do', doing: 'In progress', done: 'Done' }

export const CardSchema = z.object({
  id: z.string(),
  title: z.string().min(1),
  description: z.string(),
  column: z.enum(COLUMNS),
  label: z.string(),
  order: z.number(),
})
export type Card = z.infer<typeof CardSchema>
export type NewCard = Omit<Card, 'id'>
```

The runtime schema and the static type come from one definition, so the network boundary
is checked, not just assumed.

## Step 2 — API functions

```ts
// src/api.ts
import { z } from 'zod'
import { CardSchema, type Card, type NewCard } from './types'

const BASE = 'http://localhost:3001'

async function request<T>(path: string, schema: z.ZodType<T>, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...init?.headers },
  })
  if (!res.ok) throw new Error(`${init?.method ?? 'GET'} ${path} failed: ${res.status}`)
  return schema.parse(await res.json())
}

export const getCards = (signal?: AbortSignal) => request('/cards', z.array(CardSchema), { signal })

export const createCard = (card: NewCard) =>
  request('/cards', CardSchema, { method: 'POST', body: JSON.stringify(card) })

export const updateCard = (id: string, patch: Partial<NewCard>) =>
  request(`/cards/${id}`, CardSchema, { method: 'PATCH', body: JSON.stringify(patch) })

export const deleteCard = async (id: string) => {
  const res = await fetch(`${BASE}/cards/${id}`, { method: 'DELETE' })
  if (!res.ok) throw new Error(`DELETE failed: ${res.status}`)
}
```

## Step 3 — query hooks with optimistic moves

```ts
// src/useCards.ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import * as api from './api'
import type { Card, ColumnId } from './types'

const key = ['cards'] as const

export function useCards() {
  return useQuery({ queryKey: key, queryFn: ({ signal }) => api.getCards(signal) })
}

export function useMoveCard() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: ({ id, column, order }: { id: string; column: ColumnId; order: number }) =>
      api.updateCard(id, { column, order }),
    onMutate: async ({ id, column, order }) => {
      await qc.cancelQueries({ queryKey: key })
      const previous = qc.getQueryData<Card[]>(key)
      qc.setQueryData<Card[]>(key, old =>
        old?.map(c => (c.id === id ? { ...c, column, order } : c)),
      )
      return { previous }
    },
    onError: (_e, _vars, ctx) => {
      if (ctx?.previous) qc.setQueryData(key, ctx.previous)
    },
    onSettled: () => qc.invalidateQueries({ queryKey: key }),
  })
}

export function useCreateCard() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: api.createCard,
    onSuccess: () => qc.invalidateQueries({ queryKey: key }),
  })
}

export function useUpdateCard() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: ({ id, patch }: { id: string; patch: Partial<Omit<Card, 'id'>> }) => api.updateCard(id, patch),
    onSuccess: () => qc.invalidateQueries({ queryKey: key }),
  })
}

export function useDeleteCard() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: api.deleteCard,
    onSuccess: () => qc.invalidateQueries({ queryKey: key }),
  })
}
```

## Step 4 — UI state in a store

```ts
// src/uiStore.ts
import { create } from 'zustand'

type UiState = {
  query: string
  label: string | 'all'
  editingId: string | null
  setQuery: (q: string) => void
  setLabel: (l: string | 'all') => void
  openEditor: (id: string) => void
  closeEditor: () => void
}

export const useUi = create<UiState>(set => ({
  query: '',
  label: 'all',
  editingId: null,
  setQuery: query => set({ query }),
  setLabel: label => set({ label }),
  openEditor: id => set({ editingId: id }),
  closeEditor: () => set({ editingId: null }),
}))
```

Server data is **not** in this store — only UI state that exists nowhere else.

## Step 5 — card and column components

```tsx
// src/CardItem.tsx
import { COLUMNS, type Card, type ColumnId } from './types'
import { useUi } from './uiStore'

type Props = { card: Card; onMove: (id: string, column: ColumnId) => void }

export function CardItem({ card, onMove }: Props) {
  const openEditor = useUi(s => s.openEditor)
  const index = COLUMNS.indexOf(card.column)
  const prev = COLUMNS[index - 1]
  const next = COLUMNS[index + 1]

  return (
    <li
      className="card"
      draggable
      onDragStart={e => {
        e.dataTransfer.setData('text/plain', card.id)
        e.dataTransfer.effectAllowed = 'move'
      }}
    >
      <button className="card__title" onClick={() => openEditor(card.id)}>{card.title}</button>
      {card.label && <span className={`label label--${card.label}`}>{card.label}</span>}
      <div className="card__moves">
        {prev && (
          <button onClick={() => onMove(card.id, prev)} aria-label={`Move "${card.title}" to ${prev}`}>←</button>
        )}
        {next && (
          <button onClick={() => onMove(card.id, next)} aria-label={`Move "${card.title}" to ${next}`}>→</button>
        )}
      </div>
    </li>
  )
}
```

HTML5 drag and drop isn't keyboard- or touch-accessible, so the ← / → buttons provide an
equivalent path for everyone. (Libraries such as dnd-kit implement keyboard and touch
dragging if you want richer interactions.)

```tsx
// src/Column.tsx
import { useState } from 'react'
import { COLUMN_TITLES, type Card, type ColumnId } from './types'
import { CardItem } from './CardItem'

type Props = { id: ColumnId; cards: Card[]; onMove: (id: string, column: ColumnId) => void }

export function Column({ id, cards, onMove }: Props) {
  const [over, setOver] = useState(false)
  const headingId = `col-${id}`
  return (
    <section
      className={over ? 'column column--over' : 'column'}
      aria-labelledby={headingId}
      onDragOver={e => {
        e.preventDefault() // required to allow dropping
        setOver(true)
      }}
      onDragLeave={() => setOver(false)}
      onDrop={e => {
        e.preventDefault()
        setOver(false)
        const cardId = e.dataTransfer.getData('text/plain')
        if (cardId) onMove(cardId, id)
      }}
    >
      <h2 id={headingId}>
        {COLUMN_TITLES[id]} <span className="count">{cards.length}</span>
      </h2>
      <ul>
        {cards.map(c => <CardItem key={c.id} card={c} onMove={onMove} />)}
      </ul>
    </section>
  )
}
```

Column ids are fixed and unique on the page, so plain string ids are fine here.

## Step 6 — the board

```tsx
// src/Board.tsx
import { lazy, Suspense, useMemo } from 'react'
import { COLUMNS, type ColumnId } from './types'
import { useCards, useMoveCard } from './useCards'
import { useUi } from './uiStore'
import { Column } from './Column'
import { NewCardForm } from './NewCardForm'

const CardEditor = lazy(() => import('./CardEditor'))

export function Board() {
  const { data: cards, isPending, isError, error, refetch } = useCards()
  const move = useMoveCard()
  const query = useUi(s => s.query)
  const label = useUi(s => s.label)
  const editingId = useUi(s => s.editingId)

  const byColumn = useMemo(() => {
    const q = query.toLowerCase()
    const visible = (cards ?? []).filter(
      c => (label === 'all' || c.label === label) && c.title.toLowerCase().includes(q),
    )
    const groups: Record<ColumnId, typeof visible> = { todo: [], doing: [], done: [] }
    for (const c of visible) groups[c.column].push(c)
    for (const col of COLUMNS) groups[col].sort((a, b) => a.order - b.order)
    return groups
  }, [cards, query, label])

  if (isPending) return <p aria-busy="true">Loading board…</p>
  if (isError) return <p role="alert">{error.message} <button onClick={() => refetch()}>Retry</button></p>

  function handleMove(id: string, column: ColumnId) {
    const card = cards!.find(c => c.id === id)
    if (!card || card.column === column) return
    const maxOrder = Math.max(0, ...cards!.filter(c => c.column === column).map(c => c.order))
    move.mutate({ id, column, order: maxOrder + 1 })
  }

  return (
    <>
      {move.isError && <p role="alert">Couldn't move the card — it was put back.</p>}
      <NewCardForm />
      <div className="board">
        {COLUMNS.map(col => <Column key={col} id={col} cards={byColumn[col]} onMove={handleMove} />)}
      </div>
      {editingId && (
        <Suspense fallback={null}>
          <CardEditor cardId={editingId} />
        </Suspense>
      )}
    </>
  )
}
```

The `groups` arrays are built fresh inside `useMemo`, so sorting them in place doesn't
touch cached query data. `cards!` is safe inside `handleMove` because it's only reachable
after the `isPending`/`isError` returns.

## Step 7 — lazy editor in a native dialog

```tsx
// src/CardEditor.tsx
import { useEffect, useRef, useState } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import type { Card } from './types'
import { useDeleteCard, useUpdateCard } from './useCards'
import { useUi } from './uiStore'

export default function CardEditor({ cardId }: { cardId: string }) {
  const card = useQueryClient().getQueryData<Card[]>(['cards'])?.find(c => c.id === cardId)
  const close = useUi(s => s.closeEditor)
  const update = useUpdateCard()
  const remove = useDeleteCard()
  const ref = useRef<HTMLDialogElement>(null)
  const [title, setTitle] = useState(card?.title ?? '')
  const [description, setDescription] = useState(card?.description ?? '')

  useEffect(() => {
    ref.current?.showModal()
  }, [])

  if (!card) return null

  return (
    <dialog ref={ref} onClose={close} aria-labelledby="editor-title">
      <form
        onSubmit={e => {
          e.preventDefault()
          update.mutate({ id: card.id, patch: { title: title.trim(), description } }, { onSuccess: close })
        }}
      >
        <h2 id="editor-title">Edit card</h2>
        <label>Title <input value={title} onChange={e => setTitle(e.target.value)} required /></label>
        <label>Description <textarea value={description} onChange={e => setDescription(e.target.value)} /></label>
        {update.isError && <p role="alert">Save failed.</p>}
        <button type="submit" disabled={update.isPending}>Save</button>
        <button type="button" onClick={close}>Cancel</button>
        <button type="button" className="danger" onClick={() => remove.mutate(card.id, { onSuccess: close })}>
          Delete
        </button>
      </form>
    </dialog>
  )
}
```

Because `Board` only renders `CardEditor` while `editingId` is set, closing unmounts it
(and the dialog with it). The editor's code isn't downloaded until the first card is
opened.

## Step 8 — filter bar, new card form, app shell

```tsx
// src/NewCardForm.tsx
import { useState } from 'react'
import { useCreateCard } from './useCards'
import { useUi } from './uiStore'

export function NewCardForm() {
  const create = useCreateCard()
  const [title, setTitle] = useState('')
  const query = useUi(s => s.query)
  const setQuery = useUi(s => s.setQuery)
  const label = useUi(s => s.label)
  const setLabel = useUi(s => s.setLabel)

  return (
    <div className="toolbar">
      <form
        onSubmit={e => {
          e.preventDefault()
          if (!title.trim()) return
          create.mutate(
            { title: title.trim(), description: '', column: 'todo', label: 'ops', order: Date.now() },
            { onSuccess: () => setTitle('') },
          )
        }}
      >
        <label>New card <input value={title} onChange={e => setTitle(e.target.value)} /></label>
        <button disabled={create.isPending}>Add</button>
      </form>
      <label>Search <input type="search" value={query} onChange={e => setQuery(e.target.value)} /></label>
      <label>
        Label
        <select value={label} onChange={e => setLabel(e.target.value)}>
          <option value="all">All</option>
          <option value="ops">ops</option>
          <option value="design">design</option>
          <option value="docs">docs</option>
        </select>
      </label>
    </div>
  )
}
```

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ErrorBoundary } from 'react-error-boundary'
import { Board } from './Board'
import './index.css'

const queryClient = new QueryClient({ defaultOptions: { queries: { staleTime: 10_000 } } })

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <ErrorBoundary fallbackRender={({ error }) => <p role="alert">The board crashed: {String(error)}</p>}>
        <main>
          <h1>Kanban</h1>
          <Board />
        </main>
      </ErrorBoundary>
    </QueryClientProvider>
  </StrictMode>,
)
```

Styling is left to you; a three-column CSS grid with `.column--over` highlighting is
enough.

## How It Actually Works

Trace a drag from "To do" to "Done":

1. `dragstart` on the card stores its id in the `DataTransfer` object — browser-owned
   state outside React.
2. Hovering the Done column fires `dragover` repeatedly; calling `preventDefault()` tells
   the browser this is a valid drop target. `setOver(true)` re-renders only that column
   (its state is local), and repeated `setOver(true)` calls bail out because the value
   is unchanged.
3. `drop` reads the id and calls `handleMove`, which calls `move.mutate`.
4. `onMutate` cancels in-flight card fetches (so an older response can't overwrite the
   optimistic state), snapshots the cache, and writes the moved card into it. The
   `['cards']` query's observers are notified via `useSyncExternalStore`; `Board`
   re-renders, `useMemo` regroups because `cards` is a new array, and each `Column`
   reconciles its list by key. The card now belongs to a different `Column`'s `<ul>`, so
   React unmounts its `CardItem` from the old list and mounts a fresh one in the new list
   — keys only preserve identity **among siblings of the same parent**, never across
   parents. Any local state inside `CardItem` would reset on a move, which is one reason
   it has none.
5. The PATCH completes. `onSettled` invalidates, and a background refetch reconciles the
   cache with the server's truth. If the PATCH failed, `onError` restores the snapshot
   first.

Meanwhile, typing in the search box updates the Zustand store; `NewCardForm` and `Board`
(the subscribers to `query`) re-render, but no network request happens, because
filtering is derived from cached data.

## Exercise — extend it

1. Support **reordering within a column** by dropping onto a card (compute an `order`
   between its neighbours), with keyboard "move up/down" equivalents.
2. Add a label field to the editor and a dynamic label list derived from the data.
3. Write RTL tests for: moving a card with the keyboard buttons (mock `fetch`); rollback
   when the PATCH returns 500.
4. Replace native DnD with dnd-kit and compare the accessibility of both approaches with
   a screen reader.
5. Measure the bundle before and after the lazy editor with the visualizer from lesson 7.
