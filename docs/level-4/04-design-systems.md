---
description: "Build a React design system: design tokens, headless accessible primitives, typed variants, polymorphic and ref-forwarding components, and Storybook."
---

# 04 · Design Systems & Component Libraries

A design system is a shared set of decisions — colours, spacing, type, interaction
patterns — packaged as reusable components and documentation. In React, it's usually a
component library that every product team consumes. Done well, it makes UIs consistent,
accessible by default, and fast to build. Done badly, it's a pile of over-configurable
components everyone works around.

## Layer 1: design tokens

Tokens are named design decisions. Express them as CSS custom properties so every styling
approach can use them:

```css
/* tokens.css */
:root {
  --color-bg: #ffffff;
  --color-fg: #1b1f24;
  --color-muted: #5b636e;
  --color-accent: #2f6fed;
  --color-danger: #c62828;
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --radius-sm: 4px;
  --radius-md: 8px;
  --font-sans: system-ui, -apple-system, 'Segoe UI', sans-serif;
  --text-sm: 0.875rem;
  --text-md: 1rem;
}

[data-theme='dark'] {
  --color-bg: #0f1216;
  --color-fg: #e6e9ee;
  --color-muted: #9aa3ad;
  --color-accent: #7aa2ff;
}
```

Components reference tokens, never raw hex values. Theming (dark mode, a brand for a
second product) becomes a matter of redefining tokens. Larger systems keep tokens in a
neutral format (JSON) and generate CSS, iOS and Android outputs from one source.

## Layer 2: primitives with correct behaviour

The hardest parts of UI components are behaviour and accessibility: focus trapping in
dialogs, keyboard navigation in menus, typeahead in selects, positioning of popovers.
**Headless** libraries (such as Radix Primitives, React Aria, Headless UI, Ark UI)
implement behaviour and ARIA without styles, so you add your own look.

```jsx
// Using a headless dialog primitive (Radix-style API shown for illustration)
import * as Dialog from '@radix-ui/react-dialog'

export function ConfirmDialog({ trigger, title, description, onConfirm }) {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>{trigger}</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="ds-overlay" />
        <Dialog.Content className="ds-dialog">
          <Dialog.Title>{title}</Dialog.Title>
          <Dialog.Description>{description}</Dialog.Description>
          <div className="ds-dialog__actions">
            <Dialog.Close asChild><button className="ds-btn">Cancel</button></Dialog.Close>
            <Dialog.Close asChild>
              <button className="ds-btn ds-btn--danger" onClick={onConfirm}>Confirm</button>
            </Dialog.Close>
          </div>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  )
}
```

Check each library's docs for its exact package names and API; the approach — you own
styles, the primitive owns behaviour — is the part that transfers.

"Build vs buy": writing your own accessible combobox or date picker is weeks of work and
testing with assistive tech. Most teams should build on primitives and invest in the
styling and API layer.

## Layer 3: your components — variants, not flags

```tsx
// Button.tsx
import { forwardRef, type ComponentPropsWithoutRef } from 'react'
import { cx } from './cx'

type Variant = 'primary' | 'secondary' | 'danger' | 'ghost'
type Size = 'sm' | 'md' | 'lg'

export type ButtonProps = ComponentPropsWithoutRef<'button'> & {
  variant?: Variant
  size?: Size
  loading?: boolean
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(function Button(
  { variant = 'primary', size = 'md', loading = false, disabled, className, children, ...rest },
  ref,
) {
  return (
    <button
      ref={ref}
      className={cx('ds-btn', `ds-btn--${variant}`, `ds-btn--${size}`, className)}
      disabled={disabled || loading}
      aria-busy={loading || undefined}
      {...rest}
    >
      {loading && <span className="ds-spinner" aria-hidden="true" />}
      {children}
    </button>
  )
})
```

Design decisions worth copying:

- **Finite variants** (`variant`, `size`) as unions, mapped to classes. No `color`,
  `padding` or arbitrary style props that let consumers escape the system.
- **Pass-through native props** (`...rest`) so `type`, `onClick`, `aria-*` and `form`
  all work without being re-declared.
- **Ref forwarding** so consumers can focus the button or anchor a popover to it. In
  React 19, function components can accept `ref` as a regular prop and `forwardRef`
  becomes unnecessary; `forwardRef` still works in 19 and is required in 18, so libraries
  supporting both keep using it.
- **`className` merges** rather than replaces, for the rare legitimate layout tweak.

Libraries like `class-variance-authority` (cva) formalise the variant → class mapping,
including compound variants ("danger + ghost").

## Polymorphism: `asChild` or `as`

A "button" sometimes needs to be a link. Two common patterns:

- `as` prop: `<Button as="a" href="/pricing">` — the component renders the given tag.
  Typing this well in TypeScript is notoriously fiddly.
- `asChild` (a "Slot" pattern, as in Radix): `<Button asChild><a href="/pricing">Pricing</a></Button>` — the
  component merges its props and classes onto its single child element.

Either way, keep semantics honest: navigation should render an `<a>`.

## Documentation and testing

- **Storybook** (or Ladle, or similar) renders each component in isolation with its
  variants as "stories". It doubles as a visual test bed and as living documentation for
  designers.
- **Accessibility checks** in stories (Storybook has an a11y addon based on axe).
- **Visual regression tests** (e.g. Chromatic, Playwright screenshots) catch unintended
  style changes across dozens of components.
- **Usage guidelines** — when to use `danger` vs `secondary`, not just which props exist.

## Worked example: a `Stack` layout primitive

Layout components remove one-off margins scattered across the app:

```tsx
import type { CSSProperties, ElementType, ReactNode } from 'react'

type Space = 1 | 2 | 3 | 4
type StackProps = {
  gap?: Space
  direction?: 'row' | 'column'
  align?: CSSProperties['alignItems']
  as?: ElementType
  children: ReactNode
}

export function Stack({ gap = 2, direction = 'column', align, as: Tag = 'div', children }: StackProps) {
  return (
    <Tag
      style={{
        display: 'flex',
        flexDirection: direction,
        gap: `var(--space-${gap})`,
        alignItems: align,
      }}
    >
      {children}
    </Tag>
  )
}

// usage
<Stack gap={4}>
  <h2>Billing</h2>
  <Stack direction="row" gap={2} align="center">
    <Button>Update card</Button>
    <Button variant="ghost">View invoices</Button>
  </Stack>
</Stack>
```

Spacing values are restricted to token steps, so "8px here, 10px there" inconsistency
becomes impossible.

## Distributing the library

- In a **monorepo**, the design system is a workspace package consumed directly.
- As a **published package**, it needs a build (e.g. Vite library mode or tsup), type
  declarations, `react` and `react-dom` as **peer dependencies** (never bundled — two
  copies of React break hooks), semantic versioning and a changelog.
- **Copy-in libraries** (the model popularised by shadcn/ui) put component source
  directly into each app instead of a package, trading central updates for full
  ownership.

## How It Actually Works

CSS custom properties are resolved at **computed-value time** and inherit through the DOM
like `color` does. When `[data-theme='dark']` redefines `--color-bg` on an ancestor,
every descendant that uses `var(--color-bg)` recomputes its style — no React re-render,
no JavaScript. That's why token-based theming is effectively free at runtime compared to
passing a theme object through context and regenerating styles.

Headless primitives implement behaviour with the same tools you've used: context
(compound components share state), refs (to move focus and measure), portals
(`createPortal` renders overlays into `document.body` so `overflow: hidden` ancestors
can't clip them, while events still bubble through the React tree), and effects for
document-level listeners (Escape, outside click). The `asChild`/Slot pattern uses
`cloneElement` on the single child to merge props, composing event handlers so both
the library's and the consumer's `onClick` run.

`forwardRef` exists because `ref` historically wasn't a normal prop: React strips `ref`
from props to attach it to the element. `forwardRef` gives the component a second
argument to receive it. React 19 changed function components so `ref` is passed as a
prop, which is why the wrapper is no longer needed there.

## Common mistakes

- **Escape-hatch props** (`color`, `margin`, `style` everywhere) that dissolve
  consistency.
- **Bundling React into the library** instead of a peer dependency → "invalid hook call"
  from duplicate Reacts.
- **Building complex widgets from scratch** without the accessibility testing they need.
- **Not forwarding refs / not spreading native props** → consumers can't focus or
  configure components.
- **No documentation of *when* to use what** → every team uses components differently.

## Exercise

Build a mini design system package in your project (`src/ds/`):

1. `tokens.css` with colour, spacing, radius and type tokens plus a dark theme; a theme
   toggle that sets `data-theme` on `<html>`.
2. `Button` (variants, sizes, loading, ref, native props), `TextField` (label, hint,
   error, `useId`), `Stack`, and a `Dialog` built on a headless primitive or native
   `<dialog>`.
3. Stories for each in Storybook (or a simple `/playground` route listing all variants).
4. Run an axe check on each story and fix findings.
5. Refactor one screen of an earlier project to use only design-system components and
   tokens; count how many ad-hoc CSS rules you could delete.
