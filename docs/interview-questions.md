---
title: "React Interview Questions and Answers (180 Questions)"
description: "180 React interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the React Mastery Path."
---

# React Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free React Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup with Vite &amp; Your First Component](level-1/01-setup-vite.md)

**Q1.** Why does the course use Vite rather than Create React App?

**Answer:** CRA is deprecated and no longer recommended; Vite is fast, nearly config-free and small enough to read

For routing and server rendering the React docs point to frameworks (Level 4).

**Q2.** What does the -- in `npm create vite@latest react-lab -- --template react` do?

**Answer:** Passes --template react through npm to the Vite scaffolder

Use --template react-ts for TypeScript.

**Q3.** What does src/main.jsx do?

**Answer:** Boots React and mounts &lt;App /&gt; (via createRoot)

App.jsx is the root component; vite.config.js enables the React plugin.

**Q4.** What does the React plugin in vite.config.js provide?

**Answer:** The JSX transform and Fast Refresh

index.html is the entry point that Vite serves.

### [02 · JSX in Depth](level-1/02-jsx.md)

**Q5.** What does JSX compile to?

**Answer:** Function calls that produce plain objects

That's why most JSX 'rules' are obvious consequences.

**Q6.** What can go inside {} in JSX?

**Answer:** Expressions such as a ternary, &amp;&amp;, or .map(), not statements like if or for

Compute values above the return or use expression forms.

**Q7.** Which are the correct JSX attribute names?

**Answer:** className, htmlFor, onClick, tabIndex

aria-* and data-* keep their dashes.

**Q8.** Why use a fragment (&lt;&gt;…&lt;/&gt;) instead of a div in a table row?

**Answer:** A component returns one value, and a div inside &lt;tr&gt; would be invalid HTML; fragments add nothing to the DOM

Use &lt;Fragment key={id}&gt; when a key is needed.

### [03 · Components &amp; Props](level-1/03-components-props.md)

**Q9.** What are props?

**Answer:** A component's arguments, collected into one object passed as the first argument

The usual style destructures them in the signature.

**Q10.** When does the default `inStock = true` apply?

**Answer:** When the prop is missing or undefined, but not for null or false

It is an ordinary JavaScript default parameter.

**Q11.** What is the children prop?

**Answer:** Whatever you put between a component's opening and closing tags

It is the basis of composition.

**Q12.** How do children talk back to parents?

**Answer:** Through function props, such as onSelect

Parents decide what data a child gets; the child turns it into UI.

### [04 · State with useState](level-1/04-state-usestate.md)

**Q13.** After one click, what does n show for three `setN(n + 1)` calls starting from 0?

**Answer:** 1, because n is a constant snapshot of 0 in this render

All three calls say 'set it to 0 + 1'.

**Q14.** How do you make three increments in one click add up to 3?

**Answer:** Use updater functions: setN(prev =&gt; prev + 1)

React queues them and feeds each the previous result.

**Q15.** Why does a plain `let count = 0` fail as a counter?

**Answer:** Changing it doesn't trigger a re-render, and it resets to 0 on every render

State lives outside the function call, managed by React.

**Q16.** Why mutate-then-set (profile.city = 'Chennai'; setProfile(profile)) is a bug?

**Answer:** React compares with Object.is and sees the same object, so it may skip the update

Create a new object: setProfile({ ...profile, city: 'Chennai' }).

### [05 · Handling Events](level-1/05-events.md)

**Q17.** What is the difference between onClick={handleSave} and onClick={handleSave()}?

**Answer:** The first passes a reference; the second runs handleSave during render and passes its return value

To pass arguments, wrap in an arrow function.

**Q18.** How do you prevent a form's default submission in React?

**Answer:** Call e.preventDefault(); returning false doesn't work

The handler then reads data with FormData.

**Q19.** When does onChange fire on a text input in React?

**Answer:** On every keystroke, like the DOM input event

It doesn't wait for the field to lose focus.

**Q20.** What does e.stopPropagation() in a delete button inside a clickable card do?

**Answer:** Stops the click from also reaching the card's onOpen

Events bubble up the component tree.

### [06 · Conditional Rendering, Lists &amp; Keys](level-1/06-conditional-lists-keys.md)

**Q21.** What renders for {messages.length &amp;&amp; &lt;MessageList /&gt;} when messages is empty?

**Answer:** A literal "0"

React renders numbers; use messages.length &gt; 0 &amp;&amp;.

**Q22.** Which branching style suits cases where whole outputs differ?

**Answer:** Early return with if

Ternaries suit two small inline pieces; &amp;&amp; suits 'show this or nothing'.

**Q23.** What happens to a component's state when it returns null?

**Answer:** The component still exists and keeps its state; it just produces no DOM

Example: a Toast with no message.

**Q24.** How do you turn an array into elements?

**Answer:** With .map(), giving each element a stable key

Keys help React match items between renders.

### [07 · Forms &amp; Controlled Inputs](level-1/07-forms-controlled-inputs.md)

**Q25.** What is a controlled input?

**Answer:** React owns the value through state, and every keystroke updates state

React is then the single source of truth.

**Q26.** What happens if you give an input value without onChange?

**Answer:** It becomes read-only and React warns; use defaultValue for an uncontrolled starting value

defaultValue sets the initial value of an uncontrolled input.

**Q27.** What does `setForm(prev => ({ ...prev, [name]: value }))` allow?

**Answer:** One handler serves every field, using a computed property key from the input's name

The name attribute must match a key in the state object.

**Q28.** How do you read a checkbox in a handler?

**Answer:** e.target.checked

Text, select and textarea use e.target.value, a string.

### [08 · useEffect Basics](level-1/08-useeffect-basics.md)

**Q29.** When does an effect with the dependency array [] run?

**Answer:** After mount only (and again on remount)

Omitting the array runs it after every render; [a, b] runs when they change.

**Q30.** When does React call an effect's cleanup function?

**Answer:** Before re-running the effect with new dependencies, and on unmount

Without cleanup, every remount would leave another interval running.

**Q31.** Why does StrictMode run effects twice in development?

**Answer:** It mounts, unmounts and remounts to test that cleanup is correct

With correct cleanup, the visible behaviour is identical.

**Q32.** What is the right response to a react-hooks/exhaustive-deps warning?

**Answer:** Fix the cause; every reactive value the effect reads belongs in the array

The rule is enabled in the Vite template.

### [09 · Styling React Apps](level-1/09-styling.md)

**Q33.** What do CSS Modules (*.module.css) provide?

**Answer:** Generated unique class names, so one component's .button can't clash with another's, at zero runtime cost

You still write normal CSS.

**Q34.** What are inline styles in React good and poor for?

**Answer:** Good for truly dynamic values; poor for :hover, media queries and pseudo-elements

Numbers get px for most length properties: { margin: 8 } → 8px.

**Q35.** What is the hybrid for dynamic values?

**Answer:** Pass them through CSS custom properties, such as style={{ '--pct': ... }}

The CSS then uses var(--pct) and keeps :hover and transitions.

**Q36.** What is the problem with plain global CSS in a big app?

**Answer:** Every .btn anywhere gets the rules, so collisions creep in

It's fine for resets, typography and tokens.

### [10 · Project — Expense Tracker](level-1/10-project-expense-tracker.md)

**Q37.** Where does the list of expenses live in the tracker, and why?

**Answer:** In App, because the list, summary and form all need it

Form field values stay in ExpenseForm because nobody else cares until submit.

**Q38.** Which data is derived rather than state?

**Answer:** The filtered list, totals and breakdown, computed in App during render

Ask: who needs it, and can it be computed from something else?

**Q39.** What does monthKey("2026-09-14") return with isoDate.slice(0, 7)?

**Answer:** "2026-09"

It's used for the month filter.

**Q40.** What is the caveat with today() using toISOString()?

**Answer:** It's UTC, so near midnight it may be off by a day in some time zones

A production app would format the local date instead.

## Level 2 · Intermediate

### [01 · Composition &amp; children](level-2/01-composition-children.md)

**Q41.** What problem does composition solve compared with a 'god component'?

**Answer:** Instead of a prop for every variation, the parent passes pieces of UI

Every new design would otherwise add another prop and conditional.

**Q42.** How do you give a layout several regions?

**Answer:** Pass JSX through ordinary props like header and sidebar, plus children

JSX elements are values that can be passed around.

**Q43.** Why build ConfirmDelete out of a generic Dialog?

**Answer:** Dialog stays generic and never learns about 'delete'; the specific component is thin and readable

It's specialisation through composition.

**Q44.** What is a render prop?

**Answer:** A prop that is a function returning UI

It lets the parent control a slot that depends on the child's data.

### [02 · Lifting State &amp; Data Flow](level-2/02-lifting-state.md)

**Q45.** In which direction does React data flow?

**Answer:** Down through props

Two components can't 'reach across' to each other.

**Q46.** What fixes two panels that need to coordinate as an accordion?

**Answer:** Lift the open state to their closest common parent and pass it and callbacks down

Remove state from the children, pass values down, add state to the parent.

**Q47.** What does `setOpenId(current => (current === id ? null : id))` do?

**Answer:** Toggles: opens this panel, or closes it if already open, so only one is open

The parent stores only the id of the open panel.

**Q48.** Why is lifting state described as the most important structural decision?

**Answer:** It decides which component owns each piece of data

Wrong ownership leads to duplication and sync bugs.

### [03 · useReducer](level-2/03-usereducer.md)

**Q49.** What does a reducer take and return?

**Answer:** The current state and an action; it returns the next state

Handlers report what happened rather than how to change state.

**Q50.** What must a reducer be?

**Answer:** Pure: no mutation, API calls, randomness or reading the current time

It is a plain function outside the component.

**Q51.** How should actions be named?

**Answer:** After events ('item_added', 'filter_changed'), not setters like 'setItems'

It keeps the reducer a record of what the user did and lets one action update several fields consistently.

**Q52.** When does useReducer beat useState?

**Answer:** Several values change together, updates are complex, or you want to unit test transitions without rendering

For one or two independent values useState is simpler.

### [04 · Context](level-2/04-context.md)

**Q53.** What is prop drilling?

**Answer:** Passing a prop through several components that don't use it so a deep one can

Context lets any descendant read the value directly.

**Q54.** When is the default value passed to createContext used?

**Answer:** Only when there is no provider above the consumer

The three steps are create, provide, consume.

**Q55.** Why wrap context in a provider component and a custom hook that throws without the provider?

**Answer:** It turns 'forgot the provider' into a clear error instead of a confusing undefined

The null default plus the throwing useAuth does this.

**Q56.** Why does the AuthProvider wrap its value in useMemo?

**Answer:** So consumers don't re-render just because the provider re-rendered with a new object

The memo depends on [user]. Note the lesson says real authentication needs a server.

### [05 · Custom Hooks](level-2/05-custom-hooks.md)

**Q57.** What is a custom hook?

**Answer:** A function whose name starts with use and which calls other hooks

You share the logic, not the component.

**Q58.** What does useOnlineStatus return?

**Answer:** A boolean online value, kept up to date by online/offline listeners with cleanup

Both components then use it instead of copying the effect.

**Q59.** Does calling the same custom hook in two components share state between them?

**Answer:** No: each call gets its own state

Hooks share logic, not state.

**Q60.** Which Rules of Hooks apply?

**Answer:** Call hooks only at the top level of components or custom hooks, not in loops, conditions or nested functions

React relies on the call order being the same on every render.

### [06 · Data Fetching, Loading &amp; Errors](level-2/06-data-fetching.md)

**Q61.** Why does a single status field ('idle' | 'loading' | 'success' | 'error') beat separate isLoading and isError booleans?

**Answer:** The booleans could contradict each other

One status models states explicitly.

**Q62.** Why check res.ok after fetch?

**Answer:** fetch only rejects on network failure, so a 404 or 500 would otherwise be treated as success

The code throws Error(`Request failed: ${res.status}`).

**Q63.** What does returning () =&gt; controller.abort() from the effect achieve?

**Answer:** It cancels the in-flight request when userId changes or the component unmounts, avoiding out-of-order responses

The catch ignores AbortError because we cancelled it ourselves.

**Q64.** What were the problems of the naive useEffect fetch?

**Answer:** No loading indicator, swallowed errors, 404/500 treated as success, and nothing handles changing inputs

The lesson notes most production apps eventually hand this job to a library or framework.

### [07 · Routing with React Router](level-2/07-react-router.md)

**Q65.** What does Outlet do in a layout route?

**Answer:** Renders the matching child route, while the header stays mounted

The layout wraps all child pages.

**Q66.** What does path="products/:productId" define, and how is it read?

**Answer:** A dynamic segment, read with useParams()

For example /products/42.

**Q67.** What does &lt;Route path="*"&gt; do?

**Answer:** Catches everything else, serving as the 404 page

index marks the child shown at the parent's exact path.

**Q68.** What does NavLink's end prop do on to="/"?

**Answer:** Stops / from matching every URL

NavLink adds an active class and aria-current="page" when its route matches.

### [08 · Performance Basics: memo, useMemo, useCallback](level-2/08-performance-memo.md)

**Q69.** Which things cause a component to re-render?

**Answer:** Its own state change, its parent re-rendering, or a context it reads changing

Props change only because the parent re-rendered.

**Q70.** Is a re-render the same as a DOM update?

**Answer:** No: React re-runs your function, diffs the result, and touches the DOM only where output differs

Most re-renders are cheap.

**Q71.** What should you do before reaching for memo, useMemo or useCallback?

**Answer:** Measure with the Profiler on a production build and try structural fixes like moving state down

Dev mode is slower and double-renders in StrictMode.

**Q72.** Why move the query state into a Search component rather than Dashboard?

**Answer:** So typing doesn't re-render the expensive chart

State lives where it's used.

### [09 · Testing with Vitest &amp; Testing Library](level-2/09-testing-vitest-rtl.md)

**Q73.** What roles do Vitest and React Testing Library play?

**Answer:** Vitest runs the tests (sharing Vite's config); RTL renders components and finds elements

jsdom simulates the DOM in Node; jest-dom adds matchers like toBeInTheDocument().

**Q74.** Which query does RTL put first in priority?

**Answer:** getByRole with an accessible name

getByTestId is the last resort.

**Q75.** What do getBy, queryBy differ in when nothing matches?

**Answer:** getBy throws; queryBy returns null (for asserting something is absent)

Use getBy for elements that should be there now.

**Q76.** In the Counter test with initial={2}, what is expected after one click on Increment?

**Answer:** Count: 3

user.click is awaited, then the text is asserted.

### [10 · Project — Recipe Finder SPA](level-2/10-project-recipe-finder.md)

**Q77.** Where does the search query live in the Recipe Finder?

**Answer:** In the URL (/?q=paneer)

URL state makes searches shareable and bookmarkable.

**Q78.** What does the project's toRecipe normalisation do?

**Answer:** Converts the API's strIngredient1…20 fields into a clean ingredients array, keeping API shapes out of components

It's done once in api.js.

**Q79.** What does TheMealDB return when nothing matches?

**Answer:** { meals: null }

Handle null explicitly.

**Q80.** How are favourites persisted and shared?

**Answer:** Through a context and reducer, persisted in localStorage, with a count in the header on every page

The project reuses the useFetch and debounced-value hooks too.

## Level 3 · Advanced

### [01 · The Render &amp; Reconciliation Model](level-3/01-render-reconciliation.md)

**Q81.** Which are React's three phases?

**Answer:** Trigger, render, commit

Rendering means calling your functions, not painting pixels.

**Q82.** What happens in the render phase?

**Answer:** React calls component functions and compares with the previous output; no DOM is touched, and it can be interrupted in concurrent rendering

The commit phase applies changes and is never interrupted.

**Q83.** What happens when a span switches to an input at the same position?

**Answer:** The span and everything below it is destroyed and a new input is created

Different types produce different trees.

**Q84.** Why does React use heuristics instead of a general tree diff?

**Answer:** The general algorithm is around O(n³), too slow; heuristics based on types and keys make it linear

The two assumptions: different types give different trees; keys identify stable children.

### [02 · Client State: Zustand vs Redux Toolkit](level-3/02-state-management.md)

**Q85.** What is the key distinction that removes most of the need for a store?

**Answer:** Server state (data living on a server) versus client state (UI state existing only in the browser)

Server state is best handled by a caching library; what's left for a store is usually small.

**Q86.** How does a Zustand component avoid re-rendering on unrelated changes?

**Answer:** It passes a selector and re-renders only when the selected value changes

AddToCart selects a function that never changes, so cart updates don't re-render it.

**Q87.** Why is `useCartStore(s => ({ items: s.items, clear: s.clear }))` a problem?

**Answer:** It builds a new object every time, so it re-renders on every store change

Select separately or use useShallow.

**Q88.** Does Zustand need a provider?

**Answer:** No; the store is a module-level singleton

set merges returned partial state one level deep.

### [03 · Server State with TanStack Query](level-3/03-tanstack-query.md)

**Q89.** How does TanStack Query treat server data?

**Answer:** As a cache that can go stale, not as state you own

It adds caching, deduplication, background refresh and mutation handling.

**Q90.** What does queryKey ['todos'] mean for multiple components?

**Answer:** They share one cache entry and one request

The key must include every variable the query depends on, like a dependency array.

**Q91.** What is the difference between isPending and isFetching?

**Answer:** isPending means no data yet; isFetching means a request is in flight, including background refetches

That lets you show 'Refreshing…' over old data.

**Q92.** Why must queryFn throw on a non-OK response?

**Answer:** Otherwise failed responses would be treated as data; the query only sees errors through rejected promises

The lesson uses an res.ok check.

### [04 · Accessibility in React](level-3/04-accessibility.md)

**Q93.** Why is &lt;button type="button" onClick={save}&gt; better than &lt;div onClick={save}&gt;?

**Answer:** It's focusable, activated by Enter/Space, and announced as a button

The first rule of ARIA: don't use ARIA when a native element does the job.

**Q94.** What does alt="" mean on an image?

**Answer:** It's decorative and skipped by screen readers

Informative images need meaningful alt text.

**Q95.** Why use useId for label and input ids?

**Answer:** Hard-coded ids break when a component renders twice

It also builds aria-describedby ids.

**Q96.** What does :focus-visible do?

**Answer:** Shows the focus ring for keyboard users without showing it on mouse click

Never remove outlines without replacing them.

### [05 · Forms at Scale](level-3/05-forms-at-scale.md)

**Q97.** How does React Hook Form register inputs by default?

**Answer:** Uncontrolled, reading values from the DOM via refs, so typing doesn't re-render the whole form

That's why it scales to larger forms.

**Q98.** What does z.coerce.number() do?

**Answer:** Converts the string from an input into a number before checking it

Inputs always give strings.

**Q99.** What does .refine on a Zod object do in the lesson?

**Answer:** Adds a cross-field rule (end date on or after start) and attaches the error to endDate

The path option attaches the error to a field.

**Q100.** What is the benefit of describing the data once in a schema?

**Answer:** It validates everywhere, on client and ideally the server too

zodResolver connects the schema to React Hook Form.

### [06 · Error Boundaries &amp; Suspense](level-3/06-error-boundaries-suspense.md)

**Q101.** What happens to the app when an error is thrown during render with no error boundary?

**Answer:** The entire app unmounts, leaving a blank page

An error boundary limits the damage to its subtree.

**Q102.** Which feature still requires a class component?

**Answer:** Error boundaries (no hook equivalent)

They use getDerivedStateFromError and componentDidCatch.

**Q103.** Which errors does an error boundary NOT catch?

**Answer:** Errors in event handlers and in async code like setTimeout or promise callbacks

For event handlers use try/catch and set error state.

**Q104.** What does &lt;Suspense fallback&gt; do?

**Answer:** Shows a fallback while something inside is still loading

Both Suspense and boundaries handle 'not ready' and 'went wrong' declaratively.

### [07 · Code-Splitting &amp; Lazy Loading](level-3/07-code-splitting.md)

**Q105.** What does a dynamic import('./reports') do under Vite?

**Answer:** Becomes a separate chunk (reports-[hash].js) fetched when the line runs

A static import goes in the main bundle.

**Q106.** What does React.lazy require of the imported module?

**Answer:** A default export that is a component

For named exports map them to { default: m.Name }.

**Q107.** Why call lazy at module level, not inside a component?

**Answer:** Otherwise each render creates a new lazy type and remounts

A lazy component also needs a &lt;Suspense&gt; above it.

**Q108.** Why are routes natural places to split?

**Answer:** Users visit one page at a time

Every user would otherwise download the admin panel before the login page.

### [08 · TypeScript with React](level-3/08-typescript-react.md)

**Q109.** What does React.ReactNode mean as a prop type?

**Answer:** Anything renderable: elements, strings, numbers, null, arrays of those

It's the usual type for children.

**Q110.** How do you make a wrapper accept every native button attribute?

**Answer:** ComponentPropsWithoutRef&lt;'button'&gt; &amp; { ... }

Then disabled, onClick and aria-* all type-check.

**Q111.** What file extension holds JSX in a typed project?

**Answer:** .tsx

Start with --template react-ts; types come from @types/react and @types/react-dom.

**Q112.** What is the advantage of string-literal unions such as 'primary' | 'secondary' in props?

**Answer:** Autocomplete and rejection of typos

React.FC is no longer recommended as a default; plain functions are simpler.

### [09 · Animation](level-3/09-animation.md)

**Q113.** Which technique is called the cheapest and most robust for UI animation?

**Answer:** CSS transitions driven by a state-changed class

The element stays mounted; only a class changes.

**Q114.** Which properties are cheap to animate?

**Answer:** transform and opacity, which run on the compositor without layout

Width, height, top and left force layout work every frame.

**Q115.** Why are exit animations hard in React?

**Answer:** React removes the DOM node immediately, leaving nothing to animate, so you must keep it mounted until the exit finishes

The useDelayedUnmount hook keeps it mounted for ms before removing.

**Q116.** What does the inert attribute on a hidden drawer do?

**Answer:** Removes its contents from the tab order and accessibility tree

React 19 supports it as a boolean attribute.

### [10 · Project — Kanban Board](level-3/10-project-kanban.md)

**Q117.** Where does server data versus UI-only state live in the Kanban project?

**Answer:** Cards in TanStack Query; selected card and filters in a small Zustand store

This is the server-state versus client-state split.

**Q118.** What happens in an optimistic move if the server fails?

**Answer:** The change is rolled back

Optimistic updates show the result immediately, with rollback on failure.

**Q119.** How can cards be moved?

**Answer:** By mouse drag and drop and by keyboard buttons as an accessible alternative

Accessibility is a project requirement.

**Q120.** What does json-server provide?

**Answer:** A local REST API on db.json with GET/POST /cards and GET/PATCH/DELETE /cards/:id

Its CLI flags have changed across versions.

## Level 4 · Master

### [01 · Server Components &amp; Frameworks](level-4/01-server-components.md)

**Q121.** What is true of React Server Components?

**Answer:** They run only on the server or at build time; their code never ships to the browser, only their rendered output

RSC requires a framework or bundler integration, such as Next.js's App Router; you can't flip a switch in a plain Vite app.

**Q122.** What can a Server Component do that a Client Component cannot?

**Answer:** Read databases, files and secrets directly, and be async

The table lists the opposite set for Client Components.

**Q123.** How do you opt a module into the client in an RSC app?

**Answer:** The 'use client' directive at the top of the file

Components are Server Components by default.

**Q124.** What does the database library in a Server Component add to the browser bundle?

**Answer:** Zero bytes

The component awaits its data where the data lives, with no useEffect or loading state.

### [02 · Concurrent Rendering](level-4/02-concurrent-rendering.md)

**Q125.** What problem does concurrent rendering solve?

**Answer:** A slow render freezing the page, including the text box being typed in

React can prepare an update in the background, interrupt it, and discard it if outdated.

**Q126.** In useTransition's handleChange, which update is urgent?

**Answer:** setQuery, so the input shows the character now

The filter update inside startTransition is non-urgent and may lag.

**Q127.** What happens when the user types again before a transition render finishes?

**Answer:** React abandons the in-progress render and restarts with the latest value

isPending is true while the transition is in progress.

**Q128.** What should you do with isPending?

**Answer:** Dim stale content rather than blanking it

The demo SlowRow adds artificial slowdown to show the effect.

### [03 · Performance Profiling](level-4/03-performance-profiling.md)

**Q129.** Which Core Web Vital shows React rendering performance most directly?

**Answer:** INP (Interaction to Next Paint)

A click triggering a 300 ms render is a 300 ms+ interaction.

**Q130.** What is the difference between lab and field data?

**Answer:** Lab data is measured on your machine (Lighthouse, dev tools); field data comes from real users, e.g. via the web-vitals library

Real users on mid-range phones are the ones who matter.

**Q131.** Which Profiler view is usually the fastest way to find the culprit?

**Answer:** Ranked, which sorts components by own render time

Selecting a component shows why it rendered.

**Q132.** What does 'Props changed: onSelect' on a memo component usually signal?

**Answer:** An unstable callback is defeating memoization

It's a classic signal.

### [04 · Design Systems &amp; Component Libraries](level-4/04-design-systems.md)

**Q133.** Why should components reference tokens rather than raw hex values?

**Answer:** Theming, such as dark mode or a second brand, becomes a matter of redefining tokens

Larger systems keep tokens in a neutral format like JSON and generate CSS, iOS and Android outputs from one source.

**Q134.** What do headless libraries (Radix Primitives, React Aria, Headless UI) provide?

**Answer:** Behaviour and ARIA without styles, so you add your own look

The hard parts are focus trapping, keyboard navigation, typeahead and positioning.

**Q135.** What is a design system?

**Answer:** A shared set of decisions (colours, spacing, type, interaction patterns) packaged as reusable components and documentation

Done badly it's a pile of over-configurable components everyone works around.

**Q136.** How is dark theme applied in the tokens example?

**Answer:** [data-theme='dark'] redefines the token values

Components don't change.

### [05 · Architecture for Large Apps](level-4/05-architecture.md)

**Q137.** What is the benefit of organising by feature rather than by file type?

**Answer:** Everything about invoices lives in one folder, and deleting a feature deletes one folder

Ownership also maps cleanly to teams with CODEOWNERS.

**Q138.** What may a feature import from?

**Answer:** shared and other features' public APIs only

Each feature exposes its public API through index.ts.

**Q139.** How can the dependency rules be enforced?

**Answer:** With lint rules such as no-restricted-imports blocking deep imports into another feature

Pattern: '@/features/*/*'.

**Q140.** What should you do if invoices and customers need each other?

**Answer:** Extract the shared concept into shared or a new feature, or have app compose them

No cycles between features.

### [06 · Security in React Apps](level-4/06-security.md)

**Q141.** Why does React render '&lt;img src=x onerror=...&gt;' as literal text?

**Answer:** Strings are inserted as text nodes, so the browser never parses them as HTML

Attribute values are likewise set as values, not injected into markup.

**Q142.** When is dangerouslySetInnerHTML an XSS hole?

**Answer:** When the HTML is influenced by users, even via your own API

Sanitize with a maintained library such as DOMPurify, or avoid HTML strings.

**Q143.** Why not write your own sanitizer with regular expressions?

**Answer:** HTML parsing has many edge cases (malformed tags, encodings, SVG and MathML namespaces) that attackers exploit

Use a well-maintained sanitizer.

**Q144.** What is the risk of &lt;a href={user.website}&gt;?

**Answer:** A javascript: URL runs script on click, because React escapes text but a URL is a value the browser acts on

Validate URL schemes. Authorisation, validation and secrets belong on the server.

### [07 · Testing Strategy](level-4/07-testing-strategy.md)

**Q145.** Where does the 'testing trophy' say most effort should go?

**Answer:** Integration tests, with a broad static base, solid unit tests and thin E2E

It's a heuristic, not a law.

**Q146.** What do integration tests with MSW mock?

**Answer:** Only the network, so real fetch, TanStack Query setup and error handling all run

MSW intercepts at the network layer.

**Q147.** What should E2E tests cover?

**Answer:** A handful of critical journeys like sign up, log in and checkout

They catch routing, build config, real browser APIs and the backend contract.

**Q148.** What should you not test?

**Answer:** Implementation details, third-party libraries' own behaviour, trivial markup

Such tests break on refactors without catching real bugs.

### [08 · Building &amp; Deploying](level-4/08-deployment.md)

**Q149.** Why must you never put secrets in VITE_ variables?

**Answer:** They're replaced at build time, so the literal string ends up in the public JavaScript bundle

Changing a value also requires a rebuild; use runtime config if one build serves many environments.

**Q150.** What is the SPA fallback for?

**Answer:** Serving index.html for unknown paths so reloading /invoices/42 works with BrowserRouter

Real files like /assets/*.js are still served normally.

**Q151.** Why are hashed asset filenames served with Cache-Control: public, max-age=31536000, immutable?

**Answer:** The filename changes when content changes, so they can be cached for a year

index.html should not be cached that long.

**Q152.** Why test npm run preview before deploying?

**Answer:** Some bugs (env vars, base paths, case-sensitive imports on Linux) only appear in the build

It serves dist/ locally.

### [09 · Internationalization](level-4/09-internationalization.md)

**Q153.** Why can't you build translated sentences by concatenating fragments?

**Answer:** Word order differs between languages

Use message catalogs with interpolation.

**Q154.** What do Intl APIs provide without a library?

**Answer:** Locale-aware formatting of numbers, currency, dates, relative times, lists and plural rules

You never hand-format them.

**Q155.** Why cache Intl formatter instances?

**Answer:** Constructing Intl objects is relatively expensive

The lesson caches by `${locale}|${currency}`.

**Q156.** How many plural forms do languages have?

**Answer:** It varies: English has two, some languages one, Arabic has six categories

That's why plural rules come from Intl.PluralRules or the catalog library.

### [10 · Capstone — Team Dashboard](level-4/10-capstone.md)

**Q157.** What backend options does the capstone offer?

**Answer:** A local mock server (json-server or MSW), your own small API, or an RSC framework

Write down your choice and why in an ADR.

**Q158.** Where should filter and sort state live for the task table?

**Answer:** In the URL

It makes views shareable and bookmarkable.

**Q159.** Which non-functional requirements does the capstone set?

**Answer:** Accessibility with zero axe violations on key screens, a JS budget enforced in CI, good INP, errors contained per widget, strict TypeScript

CI runs lint, types, tests, E2E and build.

**Q160.** What is the capstone's learning approach?

**Answer:** A specification, decisions, milestones and a review checklist rather than a copy-along tutorial

Plan for a few weeks of part-time work.

## Mixed questions across all levels

**Q161.** Using this reducer (the same pure logic you would give useReducer):

```javascript
function cartReducer(state, a) {
  switch (a.type) {
    case 'added': {
      const e = state.find(i => i.id === a.item.id)
      return e ? state.map(i => i.id === a.item.id ? { ...i, qty: i.qty + 1 } : i)
               : [...state, { ...a.item, qty: 1 }]
    }
    case 'removed': return state.filter(i => i.id !== a.id)
    case 'qty_set': return state.map(i => i.id === a.id ? { ...i, qty: a.qty } : i).filter(i => i.qty > 0)
    default: throw new Error('Unknown action: ' + a.type)
  }
}
let s = []
s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
s = cartReducer(s, { type: 'added', item: { id: 2, name: 'ink', price: 50 } })
console.log(s.reduce((t, i) => t + i.price * i.qty, 0))
```
What is printed?

**Answer:** 70

pen is added twice (qty 2, 20) plus ink once (50) = 70.

**Q162.** Continuing from the previous cart (pen qty 2, ink qty 1), what is the state after `{ type: 'qty_set', id: 1, qty: 0 }`?

**Answer:** Only ink remains, because items with qty 0 are filtered out

The reducer maps the change and then filters out quantities of 0 or less. Verified by running it.

**Q163.** Why does the reducer return new arrays and objects instead of mutating?

```javascript
const before = [{ id: 1, qty: 1 }]
const after = cartReducer(before, { type: 'added', item: { id: 1 } })
console.log(before === after, before[0].qty, after[0].qty)
```

**Answer:** The new state is a different array, and the old value is untouched: false 1 2

React compares state with Object.is, so a changed result must be a new reference; printed output is `false 1 2`.

**Q164.** What does the reducer do with an unknown action type like { type: 'x' }?

**Answer:** Throws Error: Unknown action: x

The default branch throws, which surfaces typos in action names.

**Q165.** A click handler runs `setN(n + 1)` three times with n = 0 in this render. What is n after the re-render?

**Answer:** 1, because n is the constant 0 in this render's snapshot

All three calls say 'set it to 0 + 1'.

**Q166.** Using updater functions, `setN(p => p + 1)` three times in one click starting from 0 gives:

**Answer:** 3

React queues the functions and feeds each the previous result. Simulating the queue gives 3.

**Q167.** In one handler: `setN(5); setN(p => p * 2)` starting from 0. What is n afterwards?

**Answer:** 10

5 is queued as a value, then the updater doubles it. (A function followed by a value, like `setN(p => p + 1); setN(10)`, ends at 10: the last one wins.)

**Q168.** React decides whether state changed using Object.is. Which result is correct?

```javascript
console.log(Object.is(NaN, NaN), Object.is(0, -0), Object.is({}, {}))
```

**Answer:** true false false

NaN is the same as NaN, 0 and -0 differ, and two separate objects are different references. Verified by running it.

**Q169.** With `{messages.length && <MessageList />}` and an empty messages array, what appears?

**Answer:** A literal "0"

0 is a number React renders. Use messages.length &gt; 0 &amp;&amp;.

**Q170.** What does this print for tasks = [done a, not-done b, done c]?

```javascript
console.log(tasks.filter(t => t.done).map(t => t.id).join(','))
```

**Answer:** a,c

filter keeps done tasks and map takes their ids.

**Q171.** When does `useEffect(fn, [])` run?

**Answer:** After mount only (and again if remounted, such as in StrictMode in development)

An omitted array runs after every render; [a, b] re-runs when they change.

**Q172.** Which cleanup behaviour is correct for an effect that starts setInterval?

**Answer:** Return () =&gt; clearInterval(id); React runs it before re-running the effect and on unmount

Without it, each remount leaves another interval running.

**Q173.** Two components both call useOnlineStatus(). How much state is shared?

**Answer:** None: each call gets its own state; hooks share logic, not state

To share state, lift it up or use context or a store.

**Q174.** Which is the correct place for state that two sibling components both need to read and change?

**Answer:** Their closest common parent, passed down with callbacks

This is lifting state up.

**Q175.** What does the TanStack Query key ['todos', id] do?

**Answer:** Identifies a separate cache entry per id; switching back reuses the cached data

The key includes every variable the query depends on, like a dependency array.

**Q176.** What happens to an error thrown in an event handler with an error boundary above it?

**Answer:** It is not caught by the boundary; use try/catch and set error state

Boundaries catch errors during rendering, lifecycle methods and constructors, not event handlers or async code.

**Q177.** Why must lazy() be called at module level?

**Answer:** Otherwise each render creates a new lazy type and remounts the component

A lazy component also needs a &lt;Suspense&gt; boundary above it.

**Q178.** What does startTransition mark as low priority?

**Answer:** The updates inside it, so urgent input updates stay responsive and an outdated render can be abandoned

isPending is true while the transition runs.

**Q179.** Why can't you set secrets in VITE_ environment variables?

**Answer:** They are replaced at build time and end up in the public JavaScript bundle

Authorisation, validation and secrets belong on the server.

**Q180.** What is the right use of dangerouslySetInnerHTML with user-provided rich text?

**Answer:** Sanitize with a maintained library like DOMPurify, or avoid HTML strings entirely

Regex sanitizers miss many HTML edge cases.

## Practical coding questions

### Cart reducer

Write cartReducer(state, action) for a shopping cart. Actions: { type: 'added', item } (increase qty if the id exists, else add with qty 1), { type: 'removed', id }, and { type: 'qty_set', id, qty } (items with qty &lt;= 0 are removed). Unknown types must throw `Unknown action: <type>`. Never mutate state.

Starter code:

```
function cartReducer(state, action) {
  // your code
}

let s = []
s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
s = cartReducer(s, { type: 'added', item: { id: 2, name: 'ink', price: 50 } })
console.log(JSON.stringify(s))
console.log(s.reduce((t, i) => t + i.price * i.qty, 0))
s = cartReducer(s, { type: 'qty_set', id: 1, qty: 0 })
console.log(JSON.stringify(s))
```

Expected output:

```text
[{"id":1,"name":"pen","price":10,"qty":2},{"id":2,"name":"ink","price":50,"qty":1}]
70
[{"id":2,"name":"ink","price":50,"qty":1}]
```

??? success "Show a solution"

    ```
    function cartReducer(state, action) {
      switch (action.type) {
        case 'added': {
          const existing = state.find(i => i.id === action.item.id)
          return existing
            ? state.map(i => (i.id === action.item.id ? { ...i, qty: i.qty + 1 } : i))
            : [...state, { ...action.item, qty: 1 }]
        }
        case 'removed':
          return state.filter(i => i.id !== action.id)
        case 'qty_set':
          return state
            .map(i => (i.id === action.id ? { ...i, qty: action.qty } : i))
            .filter(i => i.qty > 0)
        default:
          throw new Error('Unknown action: ' + action.type)
      }
    }

    let s = []
    s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
    s = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })
    s = cartReducer(s, { type: 'added', item: { id: 2, name: 'ink', price: 50 } })
    console.log(JSON.stringify(s))
    console.log(s.reduce((t, i) => t + i.price * i.qty, 0))
    s = cartReducer(s, { type: 'qty_set', id: 1, qty: 0 })
    console.log(JSON.stringify(s))
    ```

    Because the reducer is pure, it can be unit tested without rendering, and it returns new arrays and objects so React's Object.is comparison sees the change. Verified by running the solution in Node: the output matches.

### Normalise an API response

Write toRecipe(meal) that turns a TheMealDB-style object (fields i1..i3 for ingredient names and m1..m3 for measures) into { id, ingredients: [{ name, measure }] }. Trim values and skip empty ingredient names.

Starter code:

```
function toRecipe(meal) {
  // your code
}

console.log(JSON.stringify(toRecipe({ id: '7', i1: ' rice ', m1: '1 cup', i2: '', m2: '', i3: 'salt', m3: undefined })))
```

Expected output:

```text
{"id":"7","ingredients":[{"name":"rice","measure":"1 cup"},{"name":"salt"}]}
```

??? success "Show a solution"

    ```
    function toRecipe(meal) {
      const ingredients = []
      for (let i = 1; i <= 3; i++) {
        const name = meal['i' + i]?.trim()
        const measure = meal['m' + i]?.trim()
        if (name) ingredients.push({ name, measure })
      }
      return { id: meal.id, ingredients }
    }

    console.log(JSON.stringify(toRecipe({ id: '7', i1: ' rice ', m1: '1 cup', i2: '', m2: '', i3: 'salt', m3: undefined })))
    ```

    Normalising once keeps API shapes out of components. JSON.stringify omits the undefined measure for salt. Verified by running the solution in Node: output matches.

---

More free practice: the [React Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
