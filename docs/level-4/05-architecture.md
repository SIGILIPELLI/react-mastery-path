---
description: "Structure large React codebases by feature, define public APIs and dependency rules between modules, separate data, domain and UI layers, and scale teams."
---

# 05 · Architecture for Large Apps

A 20-component app can be organised any way you like. A 2,000-component app worked on
by several teams can't: without structure, every change touches unrelated files, nobody
knows what's safe to delete, and one team's refactor breaks another team's page. This
lesson covers structure that has held up in large React codebases — and the reasoning
behind it, so you can adapt it rather than copy it.

## Organise by feature, not by file type

```text
# ❌ by type: one feature is scattered across the tree
src/
  components/   (200 files)
  hooks/        (80 files)
  reducers/
  api/
  utils/

# ✅ by feature: everything about invoices lives together
src/
  app/                 # app shell: providers, router, layout, global styles
  features/
    invoices/
      api/             # queries, mutations, API types
      components/      # InvoiceTable, InvoiceForm...
      hooks/
      model/           # pure domain logic: totals, status rules
      routes.tsx
      index.ts         # the feature's public API
    customers/
    auth/
  shared/
    ui/                # design-system components (lesson 4)
    lib/               # generic helpers: formatting, fetch client
    config/
```

When you work on invoices, you work in one folder. When you delete a feature, you delete
one folder. Ownership maps cleanly to teams (`CODEOWNERS` per feature folder).

## Public APIs and dependency rules

Each feature exposes what others may use through `index.ts`; everything else is private.

```ts
// features/invoices/index.ts
export { InvoiceSummaryCard } from './components/InvoiceSummaryCard'
export { useOutstandingBalance } from './hooks/useOutstandingBalance'
export type { Invoice } from './api/types'
```

Dependency rules (enforce them — see below):

1. `app` may import from `features` and `shared`.
2. `features` may import from `shared` and **other features' public APIs only**.
3. `shared` may import from nothing app-specific (it must be liftable into a separate
   package).
4. No cycles between features. If `invoices` and `customers` need each other, extract the
   shared concept into `shared` or a new feature, or have `app` compose them.

```js
// eslint.config.js — block deep imports into another feature
export default [
  {
    files: ['src/**/*.{ts,tsx}'],
    rules: {
      'no-restricted-imports': ['error', {
        patterns: [{
          group: ['@/features/*/*'],
          message: 'Import from the feature\'s public API (@/features/<name>) instead.',
        }],
      }],
    },
  },
]
```

Dedicated tools (eslint-plugin-boundaries, dependency-cruiser, or Nx module boundary
rules) can express the full layer rules.

## Layers inside a feature

Keep three concerns apart:

- **Data access** (`api/`): HTTP calls, query keys, response validation, mapping API
  shapes to domain types. The only place that knows URLs.
- **Domain logic** (`model/`): pure functions — pricing, validation rules, state
  machines. No React, no fetch. Trivial to unit test.
- **UI** (`components/`, `hooks/`): renders data and dispatches intents; thin.

```ts
// model/invoice.ts — pure, framework-free
export type InvoiceStatus = 'draft' | 'sent' | 'paid' | 'overdue'

export function invoiceStatus(inv: { sentAt?: string; paidAt?: string; dueDate: string }, today: string): InvoiceStatus {
  if (inv.paidAt) return 'paid'
  if (!inv.sentAt) return 'draft'
  return inv.dueDate < today ? 'overdue' : 'sent'
}
```

```tsx
// components/InvoiceRow.tsx — UI only
import { invoiceStatus } from '../model/invoice'

export function InvoiceRow({ invoice, today }: Props) {
  const status = invoiceStatus(invoice, today)
  return <tr data-status={status}>...</tr>
}
```

When the "overdue" rule changes, it changes in one tested function, not in five
components.

## State placement at scale

A decision table for any new piece of state:

| Kind | Where |
|---|---|
| Server data | a server-state cache (TanStack Query) or framework loaders/RSC |
| URL-representable (filters, tabs, pagination, selected id) | the URL |
| Form drafts | the form (RHF or local state) |
| Local UI (open/closed, hover) | component state |
| Cross-cutting client state (theme, current workspace, feature flags) | context or a small store |

Most "we need Redux" pressure disappears once server data and URL state are moved to
where they belong.

## Scaling the codebase and the team

- **Monorepo** (pnpm/npm workspaces, Turborepo, Nx): apps and shared packages (design
  system, API client, config) in one repo with atomic changes across them.
- **Micro-frontends** (independently deployed parts of one UI, e.g. via Module
  Federation): they solve an *organisational* problem — many teams needing independent
  release cycles — at a real cost in complexity, duplicated dependencies, and
  consistency. Consider them only when a modular monolith demonstrably blocks teams.
- **Feature flags** decouple deploying from releasing; long-lived branches are replaced
  by flagged code on main.
- **Architecture Decision Records (ADRs)**: short documents recording *why* a decision
  was made ("We use TanStack Query for server state because…"), so future engineers
  don't relitigate or accidentally undo it.

## Worked example: adding a feature without tangling

Task: show a customer's outstanding invoice balance on the customer detail page.

1. `invoices` exposes `useOutstandingBalance(customerId)` from its public API. Internally
   it uses the invoice query and the pure `outstandingBalance()` model function.
2. `customers` renders a slot for extra panels rather than importing invoices directly:

   ```tsx
   // features/customers/components/CustomerDetail.tsx
   export function CustomerDetail({ customerId, extraPanels }: { customerId: string; extraPanels?: React.ReactNode }) {
     ...
     return (
       <>
         <CustomerHeader customer={customer} />
         {extraPanels}
       </>
     )
   }
   ```

3. The route in `app` composes the two:

   ```tsx
   // app/routes/customer.tsx
   import { CustomerDetail } from '@/features/customers'
   import { InvoiceSummaryCard } from '@/features/invoices'

   export function CustomerRoute({ id }: { id: string }) {
     return <CustomerDetail customerId={id} extraPanels={<InvoiceSummaryCard customerId={id} />} />
   }
   ```

`customers` never depends on `invoices`; the app layer composes features. This is
Level 2's composition lesson applied at the architecture level.

## How It Actually Works

The benefits of these rules come from **coupling** — how many modules must change
together — and from how tools see your code. Bundlers build a module graph from imports;
when a feature's internals are only reachable through `index.ts`, the graph has narrow
seams, so code-splitting per route pulls in one feature's subgraph, and tree-shaking can
drop unused exports. Deep cross-feature imports create edges that pull unrelated code into
chunks and make cycles likely. Circular imports in ES modules are legal but can produce
`undefined` exports at module evaluation time, depending on which module loads first — a
class of bug that dependency rules prevent entirely.

Pure domain functions matter for more than tests: React may call render functions many
times, abandon renders (Level 4 lesson 2), and — with React Compiler — memoize
automatically. All of that is safe only when logic is pure. Pushing rules into pure
`model/` functions makes the UI layer thin enough that those guarantees hold easily.

## Common mistakes

- **A `shared/` or `common/` folder that grows into everything**, including
  feature-specific code "someone might reuse".
- **Features importing each other's internals**, creating hidden coupling.
- **Business rules inside components and effects**, duplicated across screens.
- **Adopting micro-frontends for a single team.**
- **Architecture by folder names only** — without lint rules, boundaries erode within
  weeks.

## Exercise

Refactor one of your earlier projects (the Kanban board works well):

1. Restructure into `app/`, `features/<name>/`, `shared/` with an `index.ts` public API
   per feature.
2. Add the `no-restricted-imports` rule and fix every violation.
3. Extract at least three pure domain functions into `model/` and unit test them without
   React.
4. Move every piece of state to the place the decision table suggests; list what moved
   and why.
5. Write one ADR (half a page) documenting a decision you made during the refactor.
