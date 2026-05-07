# Frontend DX Research — ACS4390 Final Project

> "We came to benchmark performance. We found that performance was never the problem. The real cost is invisible — it lives in the developer, not the browser."

---

## The Research Arc

This final presentation spans two projects. The first measured performance differences between vanilla JS and a modern framework stack. The second examined an internal framework evolution — Svelte 4 to Svelte 5 runes. Both arrived at the same conclusion independently, which is the point.

---

## Project 1 — Vanilla JS vs React + Tailwind + Zustand

### The Question

Does using a modern framework stack (React, Tailwind, Zustand) produce measurable performance gains over plain HTML/CSS/JS? Is the bundle cost justified?

### What We Measured

- Bundle size at production build
- Runtime update performance (full DOM rebuild vs virtual DOM diffing)
- State management complexity (manual localStorage vs Zustand `persist`)
- Code scaling as features were added
- Lighthouse scores (FCP, TTI, TBT)

### The Numbers

| | Vanilla | React + Tailwind + Zustand |
|---|---|---|
| JS bundle | 2.98 kB (gzip: 1.30 kB) | 199.11 kB (gzip: 63.54 kB) |
| CSS bundle | 3.03 kB | 12.77 kB |

A 66× JS bundle difference. On paper, vanilla wins.

### What the Numbers Actually Mean

The 196KB gap is not bloat. It's conventions, component systems, a state model, and a diffing engine — all shipped once, cached, and never written by hand. The real cost that doesn't show up in bundle size:

- **Vanilla mutation model:** every `addTodo`, `deleteTodo`, and `toggleTodo` requires a manual `render()` and `saveState()` call after mutating state. Miss one and the UI is wrong. Miss the other and state is lost.
- **Zustand:** `set()` handles reactivity automatically. Those calls disappear entirely.
- **Persistence:** Zustand `persist` middleware = one line. Vanilla = manual `JSON.stringify`, `localStorage.setItem`, `JSON.parse`, `localStorage.getItem`, error handling for corrupted state.
- **Scaling:** adding features in vanilla means growing `querySelector` chains, tight coupling between DOM and logic, wider bug surface. React components keep state isolated by default.

### Finding

> **The bundle is prepaid infrastructure. The overhead is invisible — it lives in the developer writing the code, not in the browser running it.**

We were measuring the wrong thing. The bottleneck was never the browser.

---

## Project 2 — Svelte 5 Runes (Europe Explorer)

### The Question

If the first project showed that frameworks win on DX over vanilla, does the same pattern hold *within* a framework's own evolution? When Svelte rewrote its reactivity model, was it really about performance — or DX again?

### The App

**Europe Explorer** — fetches all European countries from the REST Countries API. Built with Svelte 5 + Vite.

**Features:**
- Country grid with flag, capital, population
- Live search + subregion filter (both `$derived`)
- Favourites persisted to `localStorage` (via `$effect`)
- Detail view with stats and clickable border countries
- Skeleton loading state
- Document title updates on country select (`$effect`)

**Run it:**

```bash
cd demo
npm install
npm run dev
```

### Finding

The compiled output of Svelte 4 and Svelte 5 is identical. The browser sees no difference. The Svelte team scrapped their most-praised feature — the `$:` magic label syntax — for zero performance gain. Pure DX.

---

## 4MAT Presentation

### Quadrant 1 — WHY
> *Why do developers keep switching frameworks if performance differences are negligible?*

Developers keep switching. Every 3–5 years, something new wins. The stated reasons are always technical — faster rendering, smaller bundles, better architecture. Two independent projects tested that claim directly and found the same thing:

**Runtime and bundle differences are negligible on any machine made in the last decade. The real reason people switch is always developer experience.**

The 199KB React bundle felt like a problem until we looked at what it replaced: manual render cycles, hand-rolled persistence, growing querySelector chains. The "expensive" framework was cheaper. The "cheap" vanilla build had hidden costs paid in developer time and cognitive load.

Frameworks don't win because the browser runs them better. They win because developers suffer less writing them.

### Quadrant 2 — WHAT
> *What does that pattern look like across frontend history?*

| Shift | Stated reason | Actual driver |
|---|---|---|
| Vanilla JS → React | "Virtual DOM is faster" | Eliminated manual render/sync cycles, DX |
| React → Svelte | "No runtime, smaller bundle" | Negligible in practice. Less boilerplate, DX |
| Svelte 4 → Svelte 5 Runes | "Better reactivity model" | Zero performance gain. Explicit over magic, DX |

The clearest case is Svelte 5 runes. The Svelte team scrapped their most-praised feature — the `$:` magic label syntax, the thing users cited as *the* reason to choose Svelte — and replaced it with explicit rune functions. The compiled output to the browser is **identical**. No performance gain. The motivation was entirely about making the code easier to reason about and easier to share across files. Pure DX.

**The 5 Runes and their React equivalents:**

| Rune | React equivalent | What it does |
|---|---|---|
| `$state()` | `useState()` | Reactive variable — mutate directly, no setter |
| `$derived()` | `useMemo()` | Computed value, auto-tracks its own dependencies |
| `$effect()` | `useEffect()` | Side effect — no dependency array needed |
| `$props()` | Function props | Component inputs |
| `$bindable()` | *(no equivalent)* | Two-way bindable prop |

### Quadrant 3 — HOW
> *How does the DX difference show up in real code?*

**Vanilla vs React — state mutation:**

```js
// Vanilla — every mutation requires manual sync
function addTodo(text) {
  state.todos.push({ id: Date.now(), text, done: false })
  render()      // forget this = UI is wrong
  saveState()   // forget this = data is lost
}
```

```js
// Zustand — set() handles reactivity, persistence is one line
const useTodos = create(persist(
  (set) => ({
    todos: [],
    addTodo: (text) => set(s => ({ todos: [...s.todos, { id: Date.now(), text, done: false }] }))
  }),
  { name: 'todos' }
))
```

**Svelte 4 vs Svelte 5 — same feature, same browser output:**

```js
// Svelte 4 — $: is a JS label repurposed by the compiler
// Confuses TypeScript, breaks autocomplete, can't leave .svelte files
let query = '';
$: filtered = countries.filter(c =>
  c.name.common.toLowerCase().includes(query.toLowerCase())
)
```

```js
// Svelte 5 — explicit, works in plain .js files, autocompletes
let query = $state('')
let filtered = $derived(
  countries.filter(c =>
    c.name.common.toLowerCase().includes(query.toLowerCase())
  )
)
```

React requires manually declaring dependencies — get it wrong and you get stale state or infinite loops:

```js
// React — forget [favourites] = stale bug. Wrong value = infinite loop.
useEffect(() => {
  localStorage.setItem('favourites', JSON.stringify([...favourites]))
}, [favourites])
```

Svelte tracks dependencies automatically at compile time:

```js
// Svelte 5 — no array. Svelte sees favourites used inside, tracks it.
$effect(() => {
  localStorage.setItem('fav-countries', JSON.stringify([...favourites]))
})
```

Every one of these differences is invisible to the browser. All of them matter to the developer.

### Quadrant 4 — WHAT IF
> *What if this is the direction the whole industry is moving?*

Svelte 5 runes are signals — reactive primitives that track their own dependencies. Svelte didn't invent this. It joined a movement already in progress:

- **Solid.js** — built on signals from day one
- **Vue 3** — `ref()` and `reactive()` are signals
- **Angular 18+** — added signals as core primitive, deprecating Zone.js
- **Preact** — `@preact/signals` as a first-party library
- **React** — the last major holdout on VDOM diffing and manual dependency arrays

What if React is now the outlier? For a decade, React's model was the default assumption. The question is flipping: why does React still require dependency arrays in 2025? Why is direct mutation forbidden when every other major framework dropped that constraint?

And the bigger question: if runtime performance is a solved problem on modern hardware, and bundle sizes are marginal with compression and caching — **is developer experience the only meaningful competition left?**

Two projects, two different comparisons, same answer. The browser never cared. It was always about the developer.

---

## Resources

- [Svelte 5 Docs](https://svelte.dev/docs)
- [Runes announcement — svelte.dev/blog/runes](https://svelte.dev/blog/runes)
- [Rich Harris — Rethinking Reactivity (YouTube)](https://www.youtube.com/watch?v=AdNJ3fydeao)
- [REST Countries API](https://restcountries.com)
