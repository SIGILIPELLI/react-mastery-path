---
description: "Compare plain CSS, CSS Modules, inline styles, and utility-first CSS in React, apply conditional classes, and see how Vite scopes module class names."
---

# 09 · Styling React Apps

React has no opinion about styling. Anything that ends up as CSS in the browser works.
That freedom is also a trap: teams mix approaches until nobody knows where a rule lives.
This lesson compares the main options so you can pick deliberately.

## 1. Plain global CSS

```jsx
// src/main.jsx
import './index.css'
```

```css
/* index.css */
.btn { padding: 0.5rem 1rem; border-radius: 6px; border: 1px solid #ccc; }
.btn--primary { background: #2f6fed; color: white; border-color: transparent; }
```

```jsx
<button className="btn btn--primary">Save</button>
```

Importing a CSS file in Vite injects it into the page. It's **global**: every `.btn`
anywhere gets these rules. That's fine for resets, typography and design tokens, and
manageable with a naming convention such as BEM (`block__element--modifier`), but in a
big app collisions creep in.

## 2. CSS Modules

Name the file `*.module.css`:

```css
/* Button.module.css */
.button { padding: 0.5rem 1rem; border-radius: 6px; }
.primary { background: #2f6fed; color: white; }
.danger { background: #d93636; color: white; }
```

```jsx
import styles from './Button.module.css'

export function Button({ variant = 'primary', children, ...rest }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`} {...rest}>
      {children}
    </button>
  )
}
```

`styles.button` is a generated, unique class name like `_button_1x9ab_1`. Another
component's `.button` can't clash with it. You still write normal CSS — media queries,
pseudo-classes, variables — with zero runtime cost.

## 3. Inline styles

```jsx
<div style={{ width: `${progress}%`, backgroundColor: progress === 100 ? 'green' : 'steelblue' }} />
```

The `style` prop takes an object with camelCased properties; numbers get `px` added
for most length properties (`{ margin: 8 }` → `8px`). Great for truly dynamic values
computed at runtime (a progress width, a drag position). Poor for everything else:
no `:hover`, no media queries, no pseudo-elements, and styles are repeated per element.

A good hybrid is passing dynamic values through **CSS custom properties**:

```jsx
<div className={styles.bar} style={{ '--pct': `${progress}%` }} />
```

```css
.bar { width: var(--pct); transition: width 200ms ease; }
.bar:hover { filter: brightness(1.1); }
```

## 4. Utility-first CSS (e.g. Tailwind)

```jsx
<button className="rounded-md bg-blue-600 px-4 py-2 text-white hover:bg-blue-700">
  Save
</button>
```

Tailwind generates small single-purpose classes and strips unused ones at build time.
You style directly in JSX and rarely write CSS files. The trade-off is long class
strings; teams usually wrap repeated combinations in components (`<Button>`). Setup
steps change between Tailwind versions, so follow the official installation guide for
Vite rather than copying config from old blog posts.

## 5. CSS-in-JS (brief note)

Libraries like styled-components and Emotion let you write CSS in JavaScript. Runtime
CSS-in-JS injects styles while rendering, which adds cost and has friction with React
Server Components and streaming (Level 4). Many teams now prefer zero-runtime options
(CSS Modules, Tailwind, or compile-time libraries such as vanilla-extract). You'll still
meet styled-components in existing codebases, so be able to read it.

## Conditional classes

Building class strings by hand gets messy:

```jsx
className={'tab' + (active ? ' tab--active' : '') + (disabled ? ' tab--disabled' : '')}
```

A tiny helper keeps it readable:

```js
export function cx(...parts) {
  return parts.filter(Boolean).join(' ')
}
```

```jsx
<li className={cx(styles.tab, active && styles.active, disabled && styles.disabled)}>
```

The popular `clsx` package does the same (plus object syntax) in a few hundred bytes.

## Worked example: a status badge with CSS Modules

```css
/* StatusBadge.module.css */
.badge {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  padding: 0.15rem 0.6rem;
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 600;
}
.badge::before {
  content: '';
  width: 0.5rem;
  height: 0.5rem;
  border-radius: 50%;
  background: currentColor;
}
.active { background: #e6f6ec; color: #1b7a3d; }
.paused { background: #fff5e0; color: #9a6400; }
.failed { background: #fde8e8; color: #b42318; }

@media (prefers-color-scheme: dark) {
  .active { background: #12351f; color: #7ee2a1; }
  .paused { background: #3a2c0b; color: #f5c865; }
  .failed { background: #3d1414; color: #f79c9c; }
}
```

```jsx
import styles from './StatusBadge.module.css'

const labels = { active: 'Active', paused: 'Paused', failed: 'Failed' }

export default function StatusBadge({ status }) {
  return <span className={`${styles.badge} ${styles[status]}`}>{labels[status]}</span>
}
```

Everything visual lives in CSS (including dark mode and the dot), and the component only
picks a class. That separation keeps components small and designers able to tweak CSS
without touching logic.

## How It Actually Works

When Vite sees `import './index.css'`, it compiles the CSS and, in development, turns
the import into a JavaScript module that inserts a `<style>` tag into `<head>`. Editing
the file swaps that tag's contents without reloading the page. In a production build,
all imported CSS is extracted into `.css` files that the HTML links to.

For `*.module.css`, the CSS goes through a CSS Modules transform: each class selector is
rewritten to a unique hashed name, and the import's default export becomes an object
mapping original names to generated ones, e.g. `{ button: '_button_1x9ab_1' }`. Scoping
is therefore achieved by **renaming**, not by any runtime isolation. `:global(.foo)`
opts a selector out of renaming.

The `style` prop is applied by React directly to `element.style`. On updates, React
diffs the old and new style objects key by key and only sets properties that changed,
removing ones that disappeared. Custom properties (`--pct`) are set with
`style.setProperty`, which is why they work in the style object.

`className` is simply assigned to the DOM element's `className` property. React doesn't
parse it; the browser's CSS engine does the matching.

## Common mistakes

- **`class=` instead of `className=`** in JSX.
- **Kebab-case in style objects** (`{ 'font-size': 12 }`). Use `fontSize`.
- **Unitless numbers where units are needed** — React adds `px` to most lengths, but
  not to unitless properties like `lineHeight`, `opacity` or `zIndex`, which is usually
  what you want; be deliberate.
- **Referencing a class that isn't in the module** — `styles.primray` is `undefined`,
  so the class silently disappears. Typos don't error.
- **Mixing three styling approaches in one component.** Pick one primary approach per
  project and document it.

## Exercise

Style a `TaskCard` component three ways and compare:

1. Version A with a global stylesheet using BEM class names.
2. Version B with CSS Modules, including a hover state, a focus-visible outline and a
   `done` variant that strikes through the title.
3. Version C where the card's left border colour comes from a `priority` prop
   (`low`/`medium`/`high`) passed via a CSS custom property.
4. Render two cards side by side on the same page with both A and B styles present,
   and confirm in dev tools that B's class names are hashed and can't clash with A's.
5. Write three sentences on which you would choose for a 50-component app and why.
