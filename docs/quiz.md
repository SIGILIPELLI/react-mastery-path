---
title: "Final Quiz"
description: "Test what you have learned across all four levels of React Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Using this reducer (the same pure logic you would give useReducer):\n```javascript\nfunction cartReducer(state, a) {\n  switch (a.type) {\n    case 'added': {\n      const e = state.find(i => i.id === a.item.id)\n      return e ? state.map(i => i.id === a.item.id ? { ...i, qty: i.qty + 1 } : i)\n               : [...state, { ...a.item, qty: 1 }]\n    }\n    case 'removed': return state.filter(i => i.id !== a.id)\n    case 'qty_set': return state.map(i => i.id === a.id ? { ...i, qty: a.qty } : i).filter(i => i.qty > 0)\n    default: throw new Error('Unknown action: ' + a.type)\n  }\n}\nlet s = []\ns = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })\ns = cartReducer(s, { type: 'added', item: { id: 1, name: 'pen', price: 10 } })\ns = cartReducer(s, { type: 'added', item: { id: 2, name: 'ink', price: 50 } })\nconsole.log(s.reduce((t, i) => t + i.price * i.qty, 0))\n```\nWhat is printed?",
    "options": [
      "110",
      "20",
      "60",
      "70"
    ],
    "correctIndex": 3,
    "explanation": "pen is added twice (qty 2, 20) plus ink once (50) = 70."
  },
  {
    "question": "Continuing from the previous cart (pen qty 2, ink qty 1), what is the state after `{ type: 'qty_set', id: 1, qty: 0 }`?",
    "options": [
      "pen with qty 0 and ink",
      "An empty cart",
      "Only ink remains, because items with qty 0 are filtered out",
      "An error"
    ],
    "correctIndex": 2,
    "explanation": "The reducer maps the change and then filters out quantities of 0 or less. Verified by running it."
  },
  {
    "question": "Why does the reducer return new arrays and objects instead of mutating?\n```javascript\nconst before = [{ id: 1, qty: 1 }]\nconst after = cartReducer(before, { type: 'added', item: { id: 1 } })\nconsole.log(before === after, before[0].qty, after[0].qty)\n```\n",
    "options": [
      "The new state is a different array, and the old value is untouched: false 1 2",
      "true 1 2",
      "false 2 2",
      "true 2 2: it mutates in place"
    ],
    "correctIndex": 0,
    "explanation": "React compares state with Object.is, so a changed result must be a new reference; printed output is `false 1 2`."
  },
  {
    "question": "What does the reducer do with an unknown action type like { type: 'x' }?",
    "options": [
      "Returns undefined",
      "Ignores it silently",
      "Throws Error: Unknown action: x",
      "Returns the state unchanged"
    ],
    "correctIndex": 2,
    "explanation": "The default branch throws, which surfaces typos in action names."
  },
  {
    "question": "A click handler runs `setN(n + 1)` three times with n = 0 in this render. What is n after the re-render?",
    "options": [
      "0",
      "It re-renders three times with 1, 2, 3",
      "1, because n is the constant 0 in this render's snapshot",
      "3"
    ],
    "correctIndex": 2,
    "explanation": "All three calls say 'set it to 0 + 1'."
  },
  {
    "question": "Using updater functions, `setN(p => p + 1)` three times in one click starting from 0 gives:",
    "options": [
      "It depends on timing",
      "3",
      "0",
      "1"
    ],
    "correctIndex": 1,
    "explanation": "React queues the functions and feeds each the previous result. Simulating the queue gives 3."
  },
  {
    "question": "In one handler: `setN(5); setN(p => p * 2)` starting from 0. What is n afterwards?",
    "options": [
      "5",
      "Error",
      "10",
      "0"
    ],
    "correctIndex": 2,
    "explanation": "5 is queued as a value, then the updater doubles it. (A function followed by a value, like `setN(p => p + 1); setN(10)`, ends at 10: the last one wins.)"
  },
  {
    "question": "React decides whether state changed using Object.is. Which result is correct?\n```javascript\nconsole.log(Object.is(NaN, NaN), Object.is(0, -0), Object.is({}, {}))\n```\n",
    "options": [
      "true false false",
      "false true true",
      "false false false",
      "true true false"
    ],
    "correctIndex": 0,
    "explanation": "NaN is the same as NaN, 0 and -0 differ, and two separate objects are different references. Verified by running it."
  },
  {
    "question": "With `{messages.length && <MessageList />}` and an empty messages array, what appears?",
    "options": [
      "An empty list",
      "Nothing",
      "A literal \"0\"",
      "An error"
    ],
    "correctIndex": 2,
    "explanation": "0 is a number React renders. Use messages.length > 0 &&."
  },
  {
    "question": "What does this print for tasks = [done a, not-done b, done c]?\n```javascript\nconsole.log(tasks.filter(t => t.done).map(t => t.id).join(','))\n```\n",
    "options": [
      "b",
      "a,c",
      "a,b,c",
      "c,a"
    ],
    "correctIndex": 1,
    "explanation": "filter keeps done tasks and map takes their ids."
  },
  {
    "question": "When does `useEffect(fn, [])` run?",
    "options": [
      "After mount only (and again if remounted, such as in StrictMode in development)",
      "Before the first render",
      "Only when state changes",
      "After every render"
    ],
    "correctIndex": 0,
    "explanation": "An omitted array runs after every render; [a, b] re-runs when they change."
  },
  {
    "question": "Which cleanup behaviour is correct for an effect that starts setInterval?",
    "options": [
      "Call clearInterval inside the interval",
      "Return () => clearInterval(id); React runs it before re-running the effect and on unmount",
      "Cleanup only runs on page reload",
      "Do nothing; intervals stop themselves"
    ],
    "correctIndex": 1,
    "explanation": "Without it, each remount leaves another interval running."
  },
  {
    "question": "Two components both call useOnlineStatus(). How much state is shared?",
    "options": [
      "Shared in StrictMode",
      "None: each call gets its own state; hooks share logic, not state",
      "Shared only via context",
      "One shared value"
    ],
    "correctIndex": 1,
    "explanation": "To share state, lift it up or use context or a store."
  },
  {
    "question": "Which is the correct place for state that two sibling components both need to read and change?",
    "options": [
      "Each sibling, kept in sync with effects",
      "Their closest common parent, passed down with callbacks",
      "Inside a ref",
      "A global variable"
    ],
    "correctIndex": 1,
    "explanation": "This is lifting state up."
  },
  {
    "question": "What does the TanStack Query key ['todos', id] do?",
    "options": [
      "Uses one entry for all ids",
      "Disables caching",
      "Identifies a separate cache entry per id; switching back reuses the cached data",
      "Refetches on every render"
    ],
    "correctIndex": 2,
    "explanation": "The key includes every variable the query depends on, like a dependency array."
  },
  {
    "question": "What happens to an error thrown in an event handler with an error boundary above it?",
    "options": [
      "The app unmounts",
      "The boundary catches it",
      "React retries",
      "It is not caught by the boundary; use try/catch and set error state"
    ],
    "correctIndex": 3,
    "explanation": "Boundaries catch errors during rendering, lifecycle methods and constructors, not event handlers or async code."
  },
  {
    "question": "Why must lazy() be called at module level?",
    "options": [
      "It's only a lint rule",
      "Otherwise each render creates a new lazy type and remounts the component",
      "Lazy needs a class",
      "It speeds the build"
    ],
    "correctIndex": 1,
    "explanation": "A lazy component also needs a <Suspense> boundary above it."
  },
  {
    "question": "What does startTransition mark as low priority?",
    "options": [
      "All updates",
      "The updates inside it, so urgent input updates stay responsive and an outdated render can be abandoned",
      "Only effects",
      "Only network calls"
    ],
    "correctIndex": 1,
    "explanation": "isPending is true while the transition runs."
  },
  {
    "question": "Why can't you set secrets in VITE_ environment variables?",
    "options": [
      "They are ignored in production",
      "They are encrypted",
      "They need a server",
      "They are replaced at build time and end up in the public JavaScript bundle"
    ],
    "correctIndex": 3,
    "explanation": "Authorisation, validation and secrets belong on the server."
  },
  {
    "question": "What is the right use of dangerouslySetInnerHTML with user-provided rich text?",
    "options": [
      "Write a regex sanitizer",
      "Sanitize with a maintained library like DOMPurify, or avoid HTML strings entirely",
      "Use it directly",
      "Escape quotes only"
    ],
    "correctIndex": 1,
    "explanation": "Regex sanitizers miss many HTML edge cases."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Cart reducer

Write cartReducer(state, action) for a shopping cart. Actions: { type: 'added', item } (increase qty if the id exists, else add with qty 1), { type: 'removed', id }, and { type: 'qty_set', id, qty } (items with qty <= 0 are removed). Unknown types must throw `Unknown action: <type>`. Never mutate state.

**Starter code**

```javascript
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

**Expected output**

```text
[{"id":1,"name":"pen","price":10,"qty":2},{"id":2,"name":"ink","price":50,"qty":1}]
70
[{"id":2,"name":"ink","price":50,"qty":1}]
```

<details><summary>Show solution and explanation</summary>

<pre><code>function cartReducer(state, action) {
  switch (action.type) {
    case &#x27;added&#x27;: {
      const existing = state.find(i =&gt; i.id === action.item.id)
      return existing
        ? state.map(i =&gt; (i.id === action.item.id ? { ...i, qty: i.qty + 1 } : i))
        : [...state, { ...action.item, qty: 1 }]
    }
    case &#x27;removed&#x27;:
      return state.filter(i =&gt; i.id !== action.id)
    case &#x27;qty_set&#x27;:
      return state
        .map(i =&gt; (i.id === action.id ? { ...i, qty: action.qty } : i))
        .filter(i =&gt; i.qty &gt; 0)
    default:
      throw new Error(&#x27;Unknown action: &#x27; + action.type)
  }
}

let s = []
s = cartReducer(s, { type: &#x27;added&#x27;, item: { id: 1, name: &#x27;pen&#x27;, price: 10 } })
s = cartReducer(s, { type: &#x27;added&#x27;, item: { id: 1, name: &#x27;pen&#x27;, price: 10 } })
s = cartReducer(s, { type: &#x27;added&#x27;, item: { id: 2, name: &#x27;ink&#x27;, price: 50 } })
console.log(JSON.stringify(s))
console.log(s.reduce((t, i) =&gt; t + i.price * i.qty, 0))
s = cartReducer(s, { type: &#x27;qty_set&#x27;, id: 1, qty: 0 })
console.log(JSON.stringify(s))
</code></pre>

<p>Because the reducer is pure, it can be unit tested without rendering, and it returns new arrays and objects so React&#x27;s Object.is comparison sees the change. Verified by running the solution in Node: the output matches.</p>
</details>

### Normalise an API response

Write toRecipe(meal) that turns a TheMealDB-style object (fields i1..i3 for ingredient names and m1..m3 for measures) into { id, ingredients: [{ name, measure }] }. Trim values and skip empty ingredient names.

**Starter code**

```javascript
function toRecipe(meal) {
  // your code
}

console.log(JSON.stringify(toRecipe({ id: '7', i1: ' rice ', m1: '1 cup', i2: '', m2: '', i3: 'salt', m3: undefined })))
```

**Expected output**

```text
{"id":"7","ingredients":[{"name":"rice","measure":"1 cup"},{"name":"salt"}]}
```

<details><summary>Show solution and explanation</summary>

<pre><code>function toRecipe(meal) {
  const ingredients = []
  for (let i = 1; i &lt;= 3; i++) {
    const name = meal[&#x27;i&#x27; + i]?.trim()
    const measure = meal[&#x27;m&#x27; + i]?.trim()
    if (name) ingredients.push({ name, measure })
  }
  return { id: meal.id, ingredients }
}

console.log(JSON.stringify(toRecipe({ id: &#x27;7&#x27;, i1: &#x27; rice &#x27;, m1: &#x27;1 cup&#x27;, i2: &#x27;&#x27;, m2: &#x27;&#x27;, i3: &#x27;salt&#x27;, m3: undefined })))
</code></pre>

<p>Normalising once keeps API shapes out of components. JSON.stringify omits the undefined measure for salt. Verified by running the solution in Node: output matches.</p>
</details>
