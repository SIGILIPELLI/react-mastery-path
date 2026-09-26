---
description: "Learn what JSX compiles to, how to embed JavaScript expressions, the attribute naming rules, fragments, and why JSX is not HTML."
---

# 02 · JSX in Depth

JSX looks like HTML, but it is a JavaScript syntax extension. Every tag you write turns
into a function call that produces a plain object. Once you see that, most JSX "rules"
stop being rules to memorize and become obvious consequences.

## Embedding JavaScript with `{}`

Curly braces switch from markup back into JavaScript. Anything inside must be an
**expression** — something that produces a value.

```jsx
const user = { first: 'Asha', last: 'Rao', points: 1280 }

function Badge() {
  return (
    <div>
      <h2>{user.first} {user.last}</h2>
      <p>Points: {user.points.toLocaleString()}</p>
      <p>Level: {Math.floor(user.points / 500) + 1}</p>
      <p>{user.points > 1000 ? 'Gold member' : 'Member'}</p>
    </div>
  )
}
```

What you **cannot** put in braces: statements like `if`, `for`, or `const x = 1`. Those
don't produce values. Compute things above the `return`, or use expression forms
(ternary `? :`, `&&`, `.map()`).

## Attributes: camelCase and a few renames

Because JSX becomes JavaScript objects, attribute names follow DOM property naming:

| HTML | JSX |
|---|---|
| `class` | `className` |
| `for` | `htmlFor` |
| `onclick` | `onClick` |
| `tabindex` | `tabIndex` |
| `style="color: red"` | `style={{ color: 'red' }}` |

`aria-*` and `data-*` attributes keep their dashes: `aria-label="Close"`,
`data-testid="row"`.

String literals use quotes; anything else uses braces:

```jsx
<img src={photoUrl} alt="Team photo" width={320} />
<input type="checkbox" disabled={!canEdit} />
<button disabled>Always disabled</button>  {/* bare attribute means true */}
```

## Every tag must close

HTML lets you write `<br>` or `<img ...>`. JSX requires `<br />` and `<img ... />`.
Tags with children close normally: `<p>text</p>`.

## One root per return — fragments

A component returns **one** value. Two sibling elements are two values, so wrap them:

```jsx
function Row() {
  return (
    <>
      <td>Apples</td>
      <td>3</td>
    </>
  )
}
```

A `<div>` would work too, but inside a `<tr>` it would produce invalid HTML. Fragments
add nothing to the DOM. When you need a `key` on a fragment (lesson 6), use the long
form: `<Fragment key={id}>...</Fragment>` (imported from `react`).

## Comments in JSX

```jsx
return (
  <section>
    {/* This is a JSX comment */}
    <h2>Stats</h2>
  </section>
)
```

## What gets rendered

| Value in `{}` | Output |
|---|---|
| string, number | rendered as text (numbers too, including `0`) |
| `true`, `false`, `null`, `undefined` | nothing |
| array of elements | each item rendered in order (needs keys — lesson 6) |
| a plain object | **error**: "Objects are not valid as a React child" |

That `0` row matters: `{items.length && <List />}` renders a literal `0` when the list
is empty. Lesson 6 shows the fix.

## Worked example: a receipt line

```jsx
const order = {
  id: 'A-1042',
  items: [
    { name: 'Notebook', qty: 2, price: 3.5 },
    { name: 'Pen', qty: 5, price: 1.2 },
  ],
  coupon: null,
}

export default function Receipt() {
  const subtotal = order.items.reduce((sum, i) => sum + i.qty * i.price, 0)
  const discount = order.coupon ? subtotal * 0.1 : 0
  const total = subtotal - discount

  return (
    <article className="receipt">
      <h2>Order {order.id}</h2>
      <ul>
        {order.items.map(item => (
          <li key={item.name}>
            {item.qty} × {item.name} — ${(item.qty * item.price).toFixed(2)}
          </li>
        ))}
      </ul>
      {order.coupon && <p>Coupon applied: {order.coupon}</p>}
      <p style={{ fontWeight: 'bold' }}>Total: ${total.toFixed(2)}</p>
    </article>
  )
}
```

The calculation happens in normal JavaScript before `return`. JSX only *displays* the
results. Keeping logic out of markup makes both easier to read.

## How It Actually Works

The React plugin in Vite runs every `.jsx` file through a compiler (Babel or SWC)
before the browser sees it. With the modern "automatic runtime", this:

```jsx
<p className="hint">Hello {name}</p>
```

becomes roughly:

```js
import { jsx as _jsx } from 'react/jsx-runtime'
_jsx('p', { className: 'hint', children: ['Hello ', name] })
```

And a component tag:

```jsx
<Badge level={3} />
```

becomes `_jsx(Badge, { level: 3 })`. Note the difference: a lower-case tag compiles to
a **string** type (`'p'`), a capitalized one to a **variable reference** (`Badge`). That
is the whole reason components must be capitalized.

Calling `_jsx` does not create DOM nodes and does not call `Badge`. It returns a small
immutable object — a **React element** — roughly
`{ type: Badge, props: { level: 3 }, key: null }`. Elements are cheap descriptions.
React later reads these descriptions during rendering, calls function types, and turns
string types into real DOM elements.

This also explains the other rules:

- `class` is a reserved word in JavaScript objects historically, and the DOM property
  is `className`, so JSX uses that.
- Children become the `children` prop. Text and expressions inside a tag are just
  entries in that array.
- `{}` compiles to a slot in an argument list, which is why only expressions fit.
- Text is inserted with `textContent`-style APIs, not parsed as HTML. So
  `{'<b>hi</b>'}` shows the literal characters, which is a built-in defence against
  cross-site scripting (Level 4 lesson 6).

Older setups used the "classic runtime", which compiled to `React.createElement(...)`
and required `import React from 'react'` at the top of every file. You don't need that
import any more.

## Common mistakes

- **Using `class` or `for`.** React warns in the console; use `className`/`htmlFor`.
- **Passing a string to `style`.** `style="color:red"` throws. It must be an object
  with camelCased keys: `style={{ backgroundColor: 'navy' }}`.
- **Rendering an object.** `{user}` fails; render a field like `{user.first}`, or
  `{JSON.stringify(user)}` when debugging.
- **`if` inside braces.** `{if (ok) {...}}` is a syntax error. Use a ternary, `&&`, or
  compute a variable before `return`.
- **Forgetting parentheses on multi-line returns.** `return` followed by a newline
  returns `undefined` due to automatic semicolon insertion. Start the JSX on the same
  line or wrap it in `(`.

## Exercise

Build a `WeatherCard` component from this data (declared at the top of the file):

```js
const forecast = { city: 'Pune', tempC: 31, conditions: 'Humid', alerts: [] }
```

Requirements:

1. Show the city in an `<h2>` and the temperature in both °C and °F (°F = C × 9/5 + 32,
   rounded to a whole number).
2. Give the card `className="card hot"` when the temperature is above 30, otherwise
   `className="card"` — build the string with a template literal.
3. Show "No active alerts" when `alerts` is empty, otherwise a `<ul>` of them. Make sure
   an empty array never renders a stray `0`.
4. Paste your component into the Babel online REPL (or read the compiled output in
   your browser dev tools' Sources panel) and identify the `jsx(...)` calls.
