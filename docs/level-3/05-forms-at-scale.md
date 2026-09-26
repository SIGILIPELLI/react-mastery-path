---
description: "Scale React forms with React Hook Form and Zod schemas, field arrays and server errors, then compare with React 19 form actions and useActionState."
---

# 05 · Forms at Scale

Hand-written controlled forms (Level 1 lesson 7) are perfect for a login box. For a
40-field onboarding flow with conditional sections, repeating groups, async validation
and server errors, the boilerplate and re-rendering grow quickly. Two approaches help:
a form library with schema validation, and React 19's built-in form actions.

## Schema validation with Zod

Describe the data once, validate everywhere (client and, ideally, the server too):

```bash
npm install react-hook-form zod @hookform/resolvers
```

```js
// schema.js
import { z } from 'zod'

export const eventSchema = z
  .object({
    title: z.string().trim().min(3, 'At least 3 characters'),
    email: z.string().email('Enter a valid email'),
    attendees: z.coerce.number().int().min(1).max(500),
    startDate: z.string().min(1, 'Pick a date'),
    endDate: z.string().min(1, 'Pick a date'),
    speakers: z
      .array(z.object({ name: z.string().min(1, 'Name required'), topic: z.string().optional() }))
      .min(1, 'Add at least one speaker'),
  })
  .refine(d => d.endDate >= d.startDate, {
    message: 'End date must be on or after start date',
    path: ['endDate'],
  })
```

`z.coerce.number()` converts the string from an input into a number before checking it.
`refine` adds a cross-field rule and attaches its error to `endDate`. Zod's API details
have shifted between major versions (for example, some string formats gained top-level
helpers in newer releases), so confirm method names against the version you install.

## React Hook Form

React Hook Form (RHF) registers inputs **uncontrolled** by default, reading values from
the DOM through refs. Typing doesn't re-render the whole form.

```jsx
import { useForm, useFieldArray } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { eventSchema } from './schema'

export default function EventForm({ onCreate }) {
  const {
    register,
    control,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm({
    resolver: zodResolver(eventSchema),
    defaultValues: { title: '', email: '', attendees: 10, startDate: '', endDate: '', speakers: [{ name: '', topic: '' }] },
    mode: 'onTouched',
  })

  const { fields, append, remove } = useFieldArray({ control, name: 'speakers' })

  async function onSubmit(values) {
    const res = await onCreate(values)            // your API call
    if (res.fieldErrors?.title) {
      setError('title', { message: res.fieldErrors.title })  // server-side error
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <Field label="Title" error={errors.title}>
        <input {...register('title')} />
      </Field>
      <Field label="Contact email" error={errors.email}>
        <input type="email" {...register('email')} />
      </Field>
      <Field label="Attendees" error={errors.attendees}>
        <input type="number" {...register('attendees')} />
      </Field>
      <Field label="Start" error={errors.startDate}>
        <input type="date" {...register('startDate')} />
      </Field>
      <Field label="End" error={errors.endDate}>
        <input type="date" {...register('endDate')} />
      </Field>

      <fieldset>
        <legend>Speakers</legend>
        {fields.map((field, index) => (
          <div key={field.id} className="speaker-row">
            <input placeholder="Name" aria-label={`Speaker ${index + 1} name`} {...register(`speakers.${index}.name`)} />
            <input placeholder="Topic" aria-label={`Speaker ${index + 1} topic`} {...register(`speakers.${index}.topic`)} />
            <button type="button" onClick={() => remove(index)} disabled={fields.length === 1}>Remove</button>
            {errors.speakers?.[index]?.name && <p role="alert">{errors.speakers[index].name.message}</p>}
          </div>
        ))}
        <button type="button" onClick={() => append({ name: '', topic: '' })}>Add speaker</button>
        {errors.speakers?.root && <p role="alert">{errors.speakers.root.message}</p>}
      </fieldset>

      <button disabled={isSubmitting}>{isSubmitting ? 'Creating…' : 'Create event'}</button>
    </form>
  )
}

function Field({ label, error, children }) {
  return (
    <label className="field">
      <span>{label}</span>
      {children}
      {error && <span role="alert" className="error">{error.message}</span>}
    </label>
  )
}
```

Key points:

- `register('title')` returns `{ name, onChange, onBlur, ref }`, spread onto the input.
- `useFieldArray` manages repeating groups; use **`field.id`** as the key, never the index.
- `mode: 'onTouched'` validates a field after its first blur, then on every change.
- `setError` maps server-side validation back onto fields.
- For components that can't take a ref (custom selects, date pickers), RHF's
  `Controller` wraps them as controlled inputs.

Array-level errors (like "at least one speaker") surface under `errors.speakers.root` in
recent RHF versions; if you're on an older version, check where it places them.

## React 19: form actions

React 19 lets `<form action={fn}>` take a **function**. React calls it with the
`FormData` on submit, handles the pending state, and resets uncontrolled fields after
success.

```jsx
import { useActionState } from 'react'
import { useFormStatus } from 'react-dom'

async function subscribe(previousState, formData) {
  const email = String(formData.get('email') ?? '').trim()
  if (!email.includes('@')) return { error: 'Enter a valid email', email }
  const res = await fetch('/api/subscribe', { method: 'POST', body: formData })
  if (!res.ok) return { error: 'Subscription failed, try again', email }
  return { success: true }
}

function SubmitButton() {
  const { pending } = useFormStatus()           // reads the parent <form>'s status
  return <button disabled={pending}>{pending ? 'Subscribing…' : 'Subscribe'}</button>
}

export function Newsletter() {
  const [state, formAction] = useActionState(subscribe, { error: null })
  if (state.success) return <p>Thanks — check your inbox.</p>
  return (
    <form action={formAction}>
      <input name="email" type="email" defaultValue={state.email} />
      <SubmitButton />
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  )
}
```

- `useActionState(action, initialState)` returns the latest state and a wrapped action
  to pass to the form. The action receives the previous state plus the form data.
- `useFormStatus` must be called from a component **inside** the form.
- `/api/subscribe` stands in for your backend; with frameworks that support
  Server Functions (Level 4), `subscribe` could run on the server directly.
- These APIs are React 19 only. In React 18, use `onSubmit` handlers.

## Choosing

| Need | Good fit |
|---|---|
| A few fields, simple rules | plain controlled/uncontrolled inputs |
| Many fields, dynamic arrays, complex client validation | React Hook Form + Zod |
| Progressive enhancement, server-first framework | form actions (`useActionState`) |
| Shared validation client + server | a schema library (Zod or similar) regardless |

They also combine: RHF for client UX, submitting to a server action that re-validates
with the same schema.

## How It Actually Works

A controlled form stores every value in React state, so each keystroke re-renders the
component owning that state — typically the whole form. RHF avoids this by keeping
values in a mutable internal store (a plain object inside the `useForm` instance, held
in a ref) and reading input values through the DOM refs returned by `register`. The
`onChange` it attaches updates that store and runs validation if the mode requires it;
it only triggers a React re-render when something the component **subscribed to**
changes. `formState` is a proxy: RHF records which properties you destructured
(`errors`, `isSubmitting`…) and only re-renders for changes to those. `watch` and
`useWatch` opt specific fields into re-rendering.

`handleSubmit(onSubmit)` returns an event handler that prevents the default submission,
collects values from the store, runs the resolver (here, `eventSchema.safeParseAsync`
under the hood), populates `errors` and focuses the first invalid field, and calls your
function only if validation passes.

React 19 form actions work differently: when a `<form>` with a function `action` is
submitted, React prevents navigation, builds `FormData` from the form, and runs the
function inside a **transition** (Level 4 lesson 2). While it's pending, the transition
keeps the UI responsive and `useFormStatus` reports `pending: true` to children via
context. When the action resolves, the state update from `useActionState` commits, and
React resets the form's uncontrolled fields — which is why `defaultValue={state.email}` is
used to repopulate on error.

## Common mistakes

- **Index keys in field arrays** → values jump between rows after removal.
- **Mixing `register` with `value`/`onChange` props** on the same input.
- **Trusting client validation** — the server must validate again.
- **Calling `useFormStatus` in the component that renders the `<form>`** — it only sees
  a parent form.
- **Validating on every keystroke from the first character** — noisy for users; validate
  on blur or submit first.

## Exercise

1. Build a "job application" form with RHF + Zod: personal details, a repeating
   "work experience" group (company, role, from, to, with `to >= from`), a conditional
   "visa sponsorship details" section shown only when a checkbox is ticked, and a file
   input for a CV (validate type and size in the schema).
2. Simulate a server response that rejects an email as "already applied" and show it on
   the email field with `setError`.
3. Add a `console.count('form render')` and compare how often it logs versus the same form
   built with fully controlled `useState` fields.
4. If you're on React 19, rebuild the email step as a `useActionState` form and note what
   happens with JavaScript disabled in a framework that supports progressive enhancement
   versus a plain Vite app.
