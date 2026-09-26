---
description: "Build flexible React components through composition: children, slot props, render props, and compound components instead of ever-growing config props."
---

# 01 · Composition & children

A common beginner pattern is the "god component" with a prop for every variation:
`<Card title showIcon iconName footerText footerAlign hasCloseButton ...>`. Every new
design adds another prop, and the implementation becomes a wall of conditionals.
React's answer is **composition**: let the parent pass *pieces of UI* instead of
flags describing them.

## `children` as the main slot

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>
}

<Card>
  <h3>Storage</h3>
  <p>18.2 GB of 50 GB used</p>
  <progress value={18.2} max={50} />
</Card>
```

`Card` provides the frame (border, padding, shadow). The caller decides the contents.
No `title` prop, no `progress` prop — just JSX.

## Multiple named slots

When a layout has several regions, pass JSX through ordinary props:

```jsx
function PageLayout({ header, sidebar, children, footer }) {
  return (
    <div className="page">
      <header className="page__header">{header}</header>
      <aside className="page__sidebar">{sidebar}</aside>
      <main className="page__main">{children}</main>
      {footer && <footer className="page__footer">{footer}</footer>}
    </div>
  )
}

<PageLayout
  header={<Logo />}
  sidebar={<NavMenu items={menu} />}
  footer={<small>© Acme</small>}
>
  <Dashboard />
</PageLayout>
```

JSX elements are values, so they can be passed, stored in variables, or put in arrays
like any other value.

## Specialisation through composition

Instead of `<Dialog type="confirm" ...>`, build specific components *out of* a
generic one:

```jsx
function Dialog({ title, children, actions }) {
  return (
    <div role="dialog" aria-modal="true" aria-labelledby="dlg-title" className="dialog">
      <h2 id="dlg-title">{title}</h2>
      <div className="dialog__body">{children}</div>
      <div className="dialog__actions">{actions}</div>
    </div>
  )
}

function ConfirmDelete({ itemName, onCancel, onConfirm }) {
  return (
    <Dialog
      title="Delete item?"
      actions={
        <>
          <button onClick={onCancel}>Cancel</button>
          <button className="danger" onClick={onConfirm}>Delete</button>
        </>
      }
    >
      <p>
        <strong>{itemName}</strong> will be permanently removed.
      </p>
    </Dialog>
  )
}
```

`ConfirmDelete` is a thin, readable component. `Dialog` stays generic and never learns
about "delete".

## Render props: passing a function that returns UI

Sometimes the slot needs data the container owns. Pass a function instead of an element:

```jsx
function DataTable({ rows, renderRow, empty = <p>No rows.</p> }) {
  if (rows.length === 0) return empty
  return (
    <table>
      <tbody>{rows.map(row => <tr key={row.id}>{renderRow(row)}</tr>)}</tbody>
    </table>
  )
}

<DataTable
  rows={invoices}
  renderRow={inv => (
    <>
      <td>{inv.number}</td>
      <td>{inv.customer}</td>
      <td className={inv.overdue ? 'overdue' : ''}>{inv.amount}</td>
    </>
  )}
/>
```

Render props were once the main way to share logic; custom hooks (lesson 5) replaced
most of those uses. They're still the right tool when a component needs to let the
caller control *how each item looks*.

## Compound components

Some widgets are families of components that work together: `Tabs`, `Tab`, `TabPanel`.
They share state implicitly through context (lesson 4 covers context in detail):

```jsx
import { createContext, useContext, useId, useState } from 'react'

const TabsContext = createContext(null)

export function Tabs({ defaultValue, children }) {
  const [active, setActive] = useState(defaultValue)
  const baseId = useId()
  return (
    <TabsContext.Provider value={{ active, setActive, baseId }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  )
}

export function TabList({ children }) {
  return <div role="tablist">{children}</div>
}

export function Tab({ value, children }) {
  const { active, setActive, baseId } = useContext(TabsContext)
  const selected = active === value
  return (
    <button
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={selected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={selected ? 0 : -1}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  )
}

export function TabPanel({ value, children }) {
  const { active, baseId } = useContext(TabsContext)
  if (active !== value) return null
  return (
    <div role="tabpanel" id={`${baseId}-panel-${value}`} aria-labelledby={`${baseId}-tab-${value}`}>
      {children}
    </div>
  )
}
```

Usage reads like markup:

```jsx
<Tabs defaultValue="profile">
  <TabList>
    <Tab value="profile">Profile</Tab>
    <Tab value="billing">Billing</Tab>
  </TabList>
  <TabPanel value="profile"><ProfileForm /></TabPanel>
  <TabPanel value="billing"><BillingHistory /></TabPanel>
</Tabs>
```

The caller can reorder tabs, wrap them in other elements, or add an icon, and nothing
in the `Tabs` implementation changes. `useId` generates stable unique ids so multiple
tab groups on a page don't collide. (A full keyboard implementation would also handle
arrow keys; Level 3's accessibility lesson covers that.)

## Worked example: refactoring a config-prop component

Before:

```jsx
<Alert type="warning" title="Low balance" message="Top up to avoid failed payments."
       showButton buttonText="Top up" onButtonClick={openTopUp} dismissible />
```

After:

```jsx
function Alert({ tone = 'info', children, onDismiss }) {
  return (
    <div role="alert" className={`alert alert--${tone}`}>
      <div className="alert__content">{children}</div>
      {onDismiss && (
        <button className="alert__close" aria-label="Dismiss" onClick={onDismiss}>×</button>
      )}
    </div>
  )
}

<Alert tone="warning" onDismiss={hide}>
  <strong>Low balance.</strong> Top up to avoid failed payments.{' '}
  <button onClick={openTopUp}>Top up</button>
</Alert>
```

Seven props became three, and the caller can now put a link, two buttons, or a
formatted amount in the alert without another prop being added.

## How It Actually Works

`children` is not special syntax; the JSX compiler simply puts whatever is between the
tags into `props.children`. `<Card><h3/><p/></Card>` compiles to
`jsx(Card, { children: [jsx('h3', {}), jsx('p', {})] })`. A single child arrives as
that element itself; several arrive as an array; none arrives as `undefined`.

Because the parent creates the child **elements**, an important performance property
follows. In

```jsx
function Layout({ children }) {
  const [open, setOpen] = useState(false)
  return <div>{/* toggle button */}{children}</div>
}

<Layout><ExpensiveChart /></Layout>
```

when `Layout`'s own state changes, it re-renders — but the `children` prop is the *same
element object* it received last time from its parent, which didn't re-render. React
sees an identical element reference in that position and skips re-rendering
`ExpensiveChart`. Composition therefore often avoids re-renders more elegantly than
`memo` does; Level 2 lesson 8 comes back to this "lift content up" trick.

Compound components rely on the fact that context lookups walk up the **component
tree**, not the DOM tree or the JSX nesting in one file. `Tab` finds the nearest
`TabsContext.Provider` above it at render time, wherever the caller placed it.

## Common mistakes

- **Adding a boolean prop for every design variant** until the component has 20 props
  and nested ternaries. Reach for `children` or slot props.
- **Manipulating `children`** with `React.Children.map` and `cloneElement` to inject
  props. It's fragile (breaks when a child is wrapped in a fragment or another
  component). Prefer context or explicit props.
- **Using a compound child outside its parent** (`<Tab>` without `<Tabs>`) — context is
  `null` and destructuring throws. Throw a helpful error in the child instead:
  `if (!ctx) throw new Error('<Tab> must be used inside <Tabs>')`.
- **Calling a render prop as a component** (`<renderRow />`) — it's a function; call it.

## Exercise

Build an `Accordion` compound component:

1. `<Accordion>`, `<Accordion.Item value>`, `<Accordion.Header>`, `<Accordion.Panel>`
   (attach sub-components as properties: `Accordion.Item = AccordionItem`).
2. Support a `multiple` prop on `Accordion`: when false, opening one item closes the
   others; when true, any number can be open.
3. Headers are `<button>`s with `aria-expanded` and `aria-controls` using `useId`.
4. Use it to render an FAQ of five questions where one answer contains a link and
   another contains a small table — proving callers control content fully.
5. Throw a clear error if `Accordion.Item` is rendered outside an `Accordion`.
