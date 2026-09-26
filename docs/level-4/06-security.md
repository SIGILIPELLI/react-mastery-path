---
description: "Secure React apps against XSS: how JSX escaping works, dangerouslySetInnerHTML and javascript: URL risks, sanitizing, token storage, CSP and dependencies."
---

# 06 · Security in React Apps

React prevents the most common web vulnerability — cross-site scripting (XSS) — by
default. But "by default" has escape hatches, and front-end code also handles tokens,
URLs, third-party scripts and hundreds of dependencies. This lesson covers the risks that
belong to the React layer and the ones React can't solve for you.

A framing to keep: **the browser is untrusted territory**. Anything in client code can be
read and modified by the user. Authorisation, validation and secrets belong on the server.

## 1. What React escapes for you

```jsx
const comment = '<img src=x onerror="alert(document.cookie)">'
return <p>{comment}</p>
```

This renders the literal text. React inserts strings as **text nodes**, so the browser
never parses them as HTML. The same applies to attribute values: `title={userInput}`
is set as an attribute value, not injected into markup.

## 2. `dangerouslySetInnerHTML`

The escape hatch for rendering HTML strings:

```jsx
<div dangerouslySetInnerHTML={{ __html: html }} />
```

If `html` contains anything influenced by users — comments, profile bios, CMS content
editable by many people, Markdown converted to HTML, even data from your own API that
originally came from users — this is an XSS hole. The awkward name and `__html` object
are deliberate friction.

When you genuinely need to render user-provided rich text, **sanitize** it with a
well-maintained HTML sanitizer such as DOMPurify:

```jsx
import DOMPurify from 'dompurify'

function RichText({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li', 'code', 'pre', 'br'],
    ALLOWED_ATTR: ['href'],
  })
  return <div className="rich-text" dangerouslySetInnerHTML={{ __html: clean }} />
}
```

Better still, avoid HTML strings: store content as structured data (or Markdown) and
render it through React components, for example with a Markdown renderer that outputs
React elements and doesn't allow raw HTML by default.

Never write your own sanitizer with regular expressions. HTML parsing has many edge cases
(malformed tags, encodings, SVG and MathML namespaces) that attackers exploit.

## 3. Dangerous URLs

React escapes text, but a URL is a *value* the browser acts on:

```jsx
<a href={user.website}>Website</a>
```

If `user.website` is `javascript:alert(1)`, clicking the link runs script. React has
warned about `javascript:` URLs since version 16.9 and has been moving towards blocking
them, but how a given version handles them is not something to build security on —
validate the URL yourself:

```js
export function safeHref(input) {
  try {
    const url = new URL(input, window.location.origin)
    return ['http:', 'https:', 'mailto:'].includes(url.protocol) ? url.href : undefined
  } catch {
    return undefined
  }
}

<a href={safeHref(user.website)} rel="noopener noreferrer" target="_blank">Website</a>
```

The same applies to `src` on iframes, `action` on forms, and any place a URL from data
controls navigation. Parsing with `new URL()` handles tricks like leading spaces or mixed
case (`JaVaScRiPt:`) that naive `startsWith` checks miss.

## 4. Other injection paths

- **Spreading untrusted objects as props**: `<div {...userProvidedProps} />` could pass
  `dangerouslySetInnerHTML` or event-handler-like attributes. Only spread objects you
  construct.
- **Direct DOM access**: `ref.current.innerHTML = x`, `insertAdjacentHTML`,
  `document.write` bypass React's escaping entirely.
- **Server rendering**: embedding data in a `<script>` for hydration must escape `</script>`
  and similar sequences; frameworks handle this, hand-rolled SSR often doesn't.
- **`eval` / `new Function`** on anything data-derived.

## 5. Tokens and sessions

Where to keep authentication state is a trade-off:

- **`localStorage`/`sessionStorage`**: readable by any script on the page, so a single
  XSS bug leaks the token.
- **`HttpOnly`, `Secure`, `SameSite` cookies** set by the server: not readable from
  JavaScript, which removes the XSS token-theft path; you then need CSRF protection
  (`SameSite` cookies plus anti-CSRF tokens or origin checks for state-changing
  requests).
- **In-memory tokens** with a refresh flow: lost on reload, but not persisted.

A widely recommended default for browser apps is server-set `HttpOnly` cookies (often
via a backend-for-frontend), with CSRF defences. Whatever you choose, remember that XSS
lets an attacker act *as the user* inside the page even if they can't read the token.

Never put secrets in front-end code. Anything in a Vite `VITE_*` environment variable is
**inlined into the JavaScript bundle** and visible to everyone.

## 6. Content Security Policy (CSP)

A CSP header tells the browser which scripts, styles and connections are allowed:

```text
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
```

Even if an attacker injects markup, a strict CSP stops inline scripts and scripts from
other origins from running. It's set by your server/host (see the next lessons), and it
needs tuning for analytics, fonts and any inline scripts your framework emits (nonces or
hashes). Treat it as defence in depth, not a replacement for correct escaping.

## 7. Dependencies and third-party scripts

A typical React app pulls in hundreds of transitive npm packages; each runs with full
access to your page.

- Commit the lockfile; install with `npm ci` in CI.
- Run `npm audit` (or an equivalent scanner) and enable automated dependency updates
  (e.g. Dependabot/Renovate) with review.
- Be wary of typo-squatted package names and of new, little-used packages for trivial
  tasks.
- Load third-party scripts sparingly; use Subresource Integrity (`integrity=` hashes) for
  scripts from CDNs where the URL is versioned.

## Worked example: a safe user-profile card

```jsx
import DOMPurify from 'dompurify'
import { safeHref } from './safeHref'

export function ProfileCard({ profile }) {
  const website = safeHref(profile.website)
  const bioHtml = DOMPurify.sanitize(profile.bioHtml ?? '', {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'br'],
    ALLOWED_ATTR: ['href'],
  })

  return (
    <article className="profile">
      <h2>{profile.displayName}</h2>                     {/* escaped text */}
      <img src={safeHref(profile.avatarUrl)} alt="" />    {/* validated URL */}
      <div dangerouslySetInnerHTML={{ __html: bioHtml }} />{/* sanitized */}
      {website && (
        <a href={website} target="_blank" rel="noopener noreferrer">
          {new URL(website).hostname}
        </a>
      )}
    </article>
  )
}
```

Even with sanitized output, the server should also validate and store safe data —
multiple layers, because any one of them can have a bug. (DOMPurify can be configured to
also enforce safe protocols on links inside the bio; its defaults already remove
`javascript:` URLs in `href`.)

## How It Actually Works

XSS happens when attacker-controlled data is interpreted as **code** by the browser's
HTML parser or script engine. React avoids the parser: when it creates a host element
during commit, it uses `document.createElement` and sets text through text nodes
(`createTextNode` / `textContent`) and attributes through `setAttribute` or DOM properties.
None of these parse HTML, so `<script>` inside a string stays text.

`dangerouslySetInnerHTML` is implemented by assigning `element.innerHTML`, which *does*
invoke the HTML parser. `<script>` tags inserted via `innerHTML` don't execute, but event
handler attributes (`onerror`, `onload`) on elements like `<img>` or `<svg>` do — which is
why `<img src=x onerror=...>` is the classic payload.

`href` is different again: the attribute is set safely, but the browser *acts on it* when
the link is followed. A `javascript:` URL's text is evaluated as script in the page's
origin. Escaping doesn't help here because nothing is being parsed as HTML; only
validating the scheme does.

A sanitizer like DOMPurify parses the untrusted HTML in an inert context (using the
browser's own parser, so it sees exactly what the browser would), walks the resulting DOM,
removes disallowed elements and attributes, and serialises the rest. Using the real
parser is what makes it robust against the parsing quirks attackers use.

## Common mistakes

- **`dangerouslySetInnerHTML` with unsanitized user or CMS content.**
- **Trusting URLs from data** in `href`/`src`.
- **Secrets in `VITE_*` variables** or anywhere in client code.
- **Client-side-only authorisation** — hiding a button doesn't protect the API.
- **Storing long-lived tokens in `localStorage`** without weighing the XSS impact.
- **Ignoring dependency updates** for months.

## Exercise

1. Build a comment component that accepts Markdown. Render it (a) naively via a
   Markdown-to-HTML library and `dangerouslySetInnerHTML`, then try a payload such as
   `<img src=x onerror=alert(1)>` in a local test; (b) with DOMPurify; (c) with a renderer
   that outputs React elements. Explain which you'd ship.
2. Write `safeHref` with unit tests covering `https:`, relative paths, `mailto:`,
   `javascript:`, `JaVaScRiPt:`, leading whitespace and invalid input.
3. Search one of your projects for `dangerouslySetInnerHTML`, `innerHTML`, `eval`, and
   `href={` and review each usage.
4. Add a strict CSP to the preview server response (Vite's `preview.headers` option) and
   fix whatever breaks.
5. Run `npm audit` and write down which findings actually affect code that runs in the
   browser.
