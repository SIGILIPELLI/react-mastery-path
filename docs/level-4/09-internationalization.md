---
description: "Internationalize React apps: message catalogs, plurals and interpolation, Intl formatting for dates, numbers and currency, lazy locales, and RTL layouts."
---

# 09 · Internationalization

Internationalization (i18n) is preparing an app to work in many languages and regions;
localization (l10n) is the actual translating and adapting. Doing i18n early is cheap;
retrofitting it into an app full of hard-coded strings, `${count} items` concatenation
and US-formatted dates is painful. This lesson covers the engineering side.

## What actually varies by locale

- **Text**, including word order — you can't build sentences by concatenating fragments.
- **Plural rules**: English has two forms (1 item / 2 items); some languages have one
  form, others have several (Arabic has six plural categories, Russian and Polish have
  more than two).
- **Numbers**: decimal and grouping separators (`1,234.5` vs `1.234,5`), and grouping
  patterns (Indian grouping `12,34,567`).
- **Currency**: symbol placement and formatting.
- **Dates and times**: order, month names, 12/24-hour clocks, first day of the week,
  time zones.
- **Direction**: Arabic, Hebrew, Persian and Urdu are written right-to-left (RTL).
- **Sorting** and case rules.

## 1. Formatting with `Intl` — no library needed

The browser's built-in `Intl` APIs handle the formatting side:

```js
const locale = 'hi-IN'

new Intl.NumberFormat(locale).format(1234567.891)
// Hindi (India) grouping

new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(2499)
new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }).format(2499)

new Intl.DateTimeFormat('en-GB', { dateStyle: 'long' }).format(new Date(2026, 8, 26))
new Intl.DateTimeFormat('en-US', { dateStyle: 'long' }).format(new Date(2026, 8, 26))

new Intl.RelativeTimeFormat('en', { numeric: 'auto' }).format(-1, 'day')   // "yesterday"
new Intl.ListFormat('en', { type: 'conjunction' }).format(['Asha', 'Ravi', 'Meera'])
new Intl.PluralRules('ar').select(3)   // returns a plural category such as "few"
```

Run these in your browser console to see exact output for each locale — the point is you
never hand-format them. Wrap them in small helpers or hooks that read the current locale,
and cache formatter instances (constructing `Intl` objects is relatively expensive):

```js
const cache = new Map()
export function formatCurrency(amount, currency, locale) {
  const key = `${locale}|${currency}`
  if (!cache.has(key)) cache.set(key, new Intl.NumberFormat(locale, { style: 'currency', currency }))
  return cache.get(key).format(amount)
}
```

## 2. Translating text with a library

For messages you need a catalog per language plus plural/interpolation support.
Widely used React options include **react-i18next** (i18next), **FormatJS / react-intl**,
and **Lingui**. The concepts are the same; this lesson uses react-i18next syntax. Check
the library's docs for current setup details.

```bash
npm install i18next react-i18next
```

```json
// public/locales/en/common.json
{
  "greeting": "Hello, {{name}}!",
  "cart_one": "You have {{count}} item in your cart",
  "cart_other": "You have {{count}} items in your cart",
  "checkout": "Checkout"
}
```

```json
// public/locales/hi/common.json
{
  "greeting": "नमस्ते, {{name}}!",
  "cart_one": "आपकी कार्ट में {{count}} आइटम है",
  "cart_other": "आपकी कार्ट में {{count}} आइटम हैं",
  "checkout": "चेकआउट"
}
```

```js
// i18n.js
import i18n from 'i18next'
import { initReactI18next } from 'react-i18next'

const loaded = {}

async function loadLocale(lng) {
  if (loaded[lng]) return
  const res = await fetch(`/locales/${lng}/common.json`)
  i18n.addResourceBundle(lng, 'common', await res.json())
  loaded[lng] = true
}

export async function setupI18n(initial) {
  await i18n.use(initReactI18next).init({
    lng: initial,
    fallbackLng: 'en',
    ns: ['common'],
    defaultNS: 'common',
    resources: {},
    interpolation: { escapeValue: false }, // React already escapes output
  })
  await loadLocale(initial)
  if (initial !== 'en') await loadLocale('en') // fallback strings
}

export async function changeLanguage(lng) {
  await loadLocale(lng)
  await i18n.changeLanguage(lng)
  document.documentElement.lang = lng
  document.documentElement.dir = i18n.dir(lng)
}
```

```jsx
import { useTranslation } from 'react-i18next'

function CartSummary({ user, count }) {
  const { t } = useTranslation()
  return (
    <>
      <h2>{t('greeting', { name: user.firstName })}</h2>
      <p>{t('cart', { count })}</p>
      <button>{t('checkout')}</button>
    </>
  )
}
```

- `{{name}}` interpolates values — translators can move it anywhere in the sentence.
- Passing `count` makes i18next pick `cart_one`/`cart_other` (or `_few`, `_many`, etc. for
  languages that have them) using `Intl.PluralRules`.
- Only the active language (and fallback) is downloaded; the app doesn't ship every
  translation to every user.
- `escapeValue: false` is safe *because* React escapes interpolated strings when
  rendering; don't combine it with `dangerouslySetInnerHTML`.

Plugins exist for loading over HTTP and detecting the browser language
(`i18next-http-backend`, `i18next-browser-languagedetector`); the manual loader above
shows what they do.

## 3. Messages with markup

Never split a sentence into pieces around a link:

```jsx
// ❌ untranslatable word order
<p>{t('agree_prefix')} <a href="/terms">{t('terms')}</a> {t('agree_suffix')}</p>
```

Use the library's component interpolation instead:

```jsx
import { Trans } from 'react-i18next'

// "agree": "By continuing you agree to our <termsLink>Terms</termsLink>."
<Trans i18nKey="agree" components={{ termsLink: <a href="/terms" /> }} />
```

The translator controls where the link goes in the sentence.

## 4. Right-to-left layouts

Setting `dir="rtl"` on `<html>` flips text direction and the inline axis. Write CSS with
**logical properties** so layouts flip automatically:

```css
/* ❌ physical: wrong in RTL */
.card { margin-left: 16px; padding-right: 8px; text-align: left; border-left: 3px solid; }

/* ✅ logical: correct in both directions */
.card { margin-inline-start: 16px; padding-inline-end: 8px; text-align: start; border-inline-start: 3px solid; }
```

Directional icons (arrows, "back" chevrons) should mirror in RTL; logos and media controls
usually shouldn't. Test with a real RTL language, not just a flipped English UI.

## Worked example: a language switcher that persists

```jsx
import { useTranslation } from 'react-i18next'
import { changeLanguage } from './i18n'

const LANGUAGES = [
  { code: 'en', label: 'English' },
  { code: 'hi', label: 'हिन्दी' },
  { code: 'ar', label: 'العربية' },
]

export function LanguageSwitcher() {
  const { i18n } = useTranslation()
  return (
    <label>
      <span className="visually-hidden">Language</span>
      <select
        value={i18n.resolvedLanguage}
        onChange={async e => {
          const lng = e.target.value
          await changeLanguage(lng)
          try { localStorage.setItem('lang', lng) } catch {}
        }}
      >
        {LANGUAGES.map(l => (
          <option key={l.code} value={l.code} lang={l.code}>{l.label}</option>
        ))}
      </select>
    </label>
  )
}
```

```jsx
// main.jsx
const initial = localStorage.getItem('lang') ?? navigator.language.split('-')[0] ?? 'en'
await setupI18n(['en', 'hi', 'ar'].includes(initial) ? initial : 'en')
createRoot(document.getElementById('root')).render(<App />)
```

Language names are shown **in their own language** so users can find theirs, and each
`<option>` carries a `lang` attribute so screen readers pronounce it correctly. (Top-level
`await` in `main.jsx` works with Vite's default modern build targets.)

## How It Actually Works

`Intl` formatters are backed by the browser's copy of the **Unicode CLDR** (Common Locale
Data Repository) data — the same dataset operating systems use — which is why output is
correct for hundreds of locales without shipping any data yourself. `Intl.PluralRules`
encodes CLDR plural rules: for a number and a locale it returns a category (`zero`,
`one`, `two`, `few`, `many`, `other`), and i18n libraries map that category to a key
suffix or an ICU `plural` branch.

`useTranslation` subscribes the component to the i18next instance's `languageChanged`
event (and resource loading events). Calling `changeLanguage` updates the instance and
triggers those listeners; subscribed components re-render and call `t` again, which
looks up keys in the new language's resource bundle with fallback to `fallbackLng`. The
`t` function itself does no React magic — it's a lookup plus interpolation.

Setting `dir` on `<html>` changes the **inline base direction**. Logical CSS properties
are defined relative to the writing mode and direction (`inline-start` is left in LTR
and right in RTL), so the browser maps them to physical sides at computed-value time.
Flexbox and grid also follow the inline direction, which is why most layouts flip without
extra CSS when you use logical properties.

## Common mistakes

- **String concatenation** (`'You have ' + n + ' items'`) — untranslatable and wrong for
  plurals.
- **Hand-formatting numbers and dates** (`toFixed(2)`, `MM/DD/YYYY`).
- **Splitting sentences around links or bold text** instead of component interpolation.
- **Physical CSS properties** everywhere → broken RTL.
- **Bundling every language into the main bundle.**
- **Forgetting `lang`/`dir` on `<html>`** → wrong screen reader pronunciation and
  direction.
- **Assuming text length** — German and Finnish strings are often much longer than
  English; design for expansion.

## Exercise

1. Internationalize the Level 1 expense tracker: extract every string into
   `en`/`hi` (or any two languages you can check with a native speaker or reliable
   source) catalogs.
2. Replace all number/currency/date formatting with cached `Intl` formatters that follow
   the current language.
3. Add correct pluralisation to "N expenses" and check it in a language with more than
   two plural forms (for example Polish or Arabic) using `Intl.PluralRules`.
4. Add Arabic (machine translations are acceptable if clearly marked as test-only, for
   testing) and fix the layout for RTL using logical properties.
5. Lazy-load catalogs so switching language downloads only that language's file — verify
   in the Network panel.
