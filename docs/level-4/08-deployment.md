---
description: "Deploy React apps: production builds, environment variables, SPA fallbacks, cache headers for hashed assets, preview deploys, CI pipelines and monitoring."
---

# 08 · Building & Deploying

Shipping a React app is mostly about static files and HTTP caching — unless you use a
server-rendering framework, in which case you also deploy a server or serverless
functions. This lesson covers the client-side SPA path in detail and outlines what
changes for frameworks.

Deploying to a hosting provider needs an account with that provider; the steps below
describe what to configure rather than any one provider's UI, which changes often.
Everything up to the upload can be practised locally.

## 1. The production build

```bash
npm run build     # outputs dist/
npm run preview   # serves dist/ locally to test the real build
```

What the build does:

- Compiles JSX/TypeScript, bundles and **minifies** JavaScript and CSS.
- Sets `process.env.NODE_ENV`/`import.meta.env.MODE` to production, so React's
  development-only checks and warnings are stripped.
- Splits code at dynamic imports and emits files with **content hashes**
  (`assets/index-4f9a1c.js`).
- Rewrites `index.html` to reference the hashed files.

Always test `npm run preview` before deploying: some bugs (environment variables, base
paths, case-sensitive imports on Linux servers) only appear in the build.

## 2. Environment variables

Vite exposes variables prefixed with `VITE_` via `import.meta.env`:

```bash
# .env.production
VITE_API_URL=https://api.example.com
```

```js
const res = await fetch(`${import.meta.env.VITE_API_URL}/invoices`)
```

These are **replaced at build time** — the literal string ends up in the JavaScript
bundle. Two consequences:

1. **Never put secrets in them.** Anyone can read the bundle.
2. Changing a value requires a **rebuild**. If you need one build deployed to many
   environments, load configuration at runtime instead (e.g. fetch `/config.json`
   served per environment, or inject a small script tag at deploy time).

## 3. SPA fallback (client-side routing)

With `BrowserRouter`, a user who reloads `/invoices/42` requests that path from the
server, which has no such file. Configure the host to serve `index.html` for unknown
paths (while still serving real files like `/assets/*.js` normally). Every serious static
host has a setting for this — look for "rewrites", "redirects" or "SPA fallback".

A generic nginx configuration:

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;

  location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri =404;
  }

  location / {
    add_header Cache-Control "no-cache";
    try_files $uri /index.html;
  }
}
```

Hosts without rewrite support (for example GitHub Pages) need workarounds such as
`HashRouter` or a copied `404.html`.

If the app is served from a sub-path (`https://example.com/app/`), set `base: '/app/'` in
`vite.config.js` and the router's `basename` to match.

## 4. Caching strategy

The content-hash file names enable the ideal caching setup:

| File | Header | Why |
|---|---|---|
| `/assets/*-[hash].js`, `.css`, fonts, images | `Cache-Control: public, max-age=31536000, immutable` | the name changes whenever content changes, so caching "forever" is safe |
| `index.html` | `Cache-Control: no-cache` (revalidate every time) | it's the entry point that points to the current hashes |

With this, users download new code the moment you deploy (fresh `index.html`) and never
re-download unchanged chunks.

Plan for **stale tabs**: a user with yesterday's `index.html` open may request a chunk
that your new deploy removed. Keep previous assets available for a while (many hosts do
this automatically with atomic, immutable deploys) and use an error boundary that
offers a reload on chunk-load failure (Level 3 lesson 7).

## 5. CI/CD pipeline

A typical pipeline on every pull request and on `main`:

```yaml
# .github/workflows/ci.yml  (GitHub Actions example)
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npx vitest run
      - run: npm run build
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist
```

Action versions and the Node version shown are examples; use current ones. The deploy
step itself depends on your host (a provider's CLI or action, or an upload to object
storage + CDN invalidation of `index.html`).

**Preview deployments** — a unique URL per pull request — are offered by most modern
hosts and let reviewers try changes before merging.

## 6. After deploy: know when it breaks

- **Error monitoring** (e.g. Sentry or similar): report errors from error boundaries'
  `componentDidCatch`/`onError` and from `window.onerror`/`unhandledrejection`. Upload
  **source maps** to the monitoring service (not publicly, if you'd rather not expose
  source) so stack traces point at your real code.
- **Real-user performance**: the `web-vitals` reporting from lesson 3.
- **Uptime checks** on a key route.
- **Rollback**: know how to redeploy the previous build in one step. Immutable, atomic
  deploys make this trivial.

## What changes with a server-rendering framework

- The build outputs server code as well as static assets; the host must run Node (or an
  edge/serverless runtime the framework supports).
- Environment variables split into server-only (secrets are fine there) and
  public/client ones — each framework has a naming convention; learn it.
- Caching includes HTML responses and data caches, not just assets.
- Many frameworks can still produce a fully static export for sites without dynamic
  server needs.

## Worked example: a Docker image for the SPA

When your organisation deploys containers:

```dockerfile
# Dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

```bash
docker build --build-arg VITE_API_URL=https://api.example.com -t my-spa .
docker run -p 8080:80 my-spa
# open http://localhost:8080/invoices/42 — the fallback serves the app
```

The multi-stage build keeps Node and `node_modules` out of the final image, which
contains only nginx and static files. `nginx.conf` is the configuration from section 3.

## How It Actually Works

When a browser loads your app, it requests `index.html`. With `no-cache`, it may keep a
copy but must **revalidate** with the server first (using `ETag`/`Last-Modified`); the
server answers `304 Not Modified` if unchanged — cheap — or sends the new file. The new
`index.html` references new hashed file names, which the browser has never seen, so it
fetches them. Files whose content didn't change keep their old hashes and are served
straight from the browser cache without any request at all, thanks to `immutable` and a
long `max-age`.

The hash in each file name is computed from the file's content (and, for chunks, from
the content of what they import — changing a dependency changes the importing chunk's
hash too, so stale cross-references can't happen).

The SPA fallback works because routing happens entirely in JavaScript: whatever the
path, the server returns the same `index.html`, the app boots, reads
`window.location.pathname`, and the router renders the matching route. The server's
only job is to not return 404 for paths that are "real" in the app.

## Common mistakes

- **Secrets in `VITE_*` variables.**
- **Long-caching `index.html`** → users stuck on old versions after deploys.
- **Short-caching hashed assets** → wasted bandwidth and slower repeat visits.
- **Missing SPA fallback** → deep links and reloads 404.
- **Deploying without running the production build locally first.**
- **No error monitoring** → you learn about crashes from users, if at all.

## Exercise

1. Build one of your apps, serve it with `npm run preview`, and confirm deep links work.
2. Write the nginx configuration and Dockerfile from this lesson for it; run the container
   and use dev tools' Network panel to verify `Cache-Control` headers on `index.html` and
   an asset, and that a reload of an asset shows it served from cache.
3. Change one component, rebuild, and note which hashed file names changed and which
   didn't.
4. Write a CI workflow for your repository that lints, type-checks, tests and builds.
5. Add a runtime `config.json` loaded before the app renders, so the same build can point
   at different API URLs.
