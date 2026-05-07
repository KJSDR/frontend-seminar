# Svelte 5 Runes — ACS4390 Final Project

**Europe Explorer** — a country browsing app built with Svelte 5, structured as a 4MAT presentation examining Svelte's runes reactivity system and the broader argument that frontend framework evolution is a developer experience competition, not a performance competition.

---

## Quadrant 1 — WHY *(Motivation)*
> *Why should anyone care about this?*

Every major frontend framework shift has been marketed as a performance or architectural improvement. The evidence says otherwise — the real driver is always developer experience (DX).

| Shift | Stated reason | Actual driver |
|---|---|---|
| Vanilla JS → React | "Virtual DOM is faster" | Component model, reusability, DX |
| React → Svelte | "No runtime, smaller bundle" | Negligible in practice. Less boilerplate, DX |
| Svelte 4 → Svelte 5 Runes | "Better reactivity model" | Zero performance gain. Explicit over magic, DX |

The strongest evidence: Svelte 5 runes. The Svelte team scrapped their most-praised feature — the `$:` magic label syntax, the thing users cited as *the* reason to choose Svelte — and replaced it with explicit rune functions. The compiled output to the browser is **identical**. The motivation was entirely about making the code easier to reason about, easier for editors to understand, and easier to share across files. Pure DX.

This project builds on an earlier finding (vanilla JS/CSS/HTML vs React + Tailwind + Zustand) that concluded runtime and bundle differences are negligible on any modern hardware. **The delta is always in the developer's chair, not the browser.**

---

## Quadrant 2 — WHAT *(Concepts)*
> *What is it? The facts, definitions, and theory.*

### What is Svelte?

Svelte is a compiler, not a runtime library. Your Svelte code compiles to vanilla DOM manipulation at build time — no virtual DOM, no diffing, no framework shipped to the browser. React ships ~45kb of runtime to every user. Svelte ships near zero.

### The 5 Runes

| Rune | React equivalent | What it does |
|---|---|---|
| `$state()` | `useState()` | Reactive variable — mutate directly, no setter function |
| `$derived()` | `useMemo()` | Computed value, auto-updates when deps change |
| `$effect()` | `useEffect()` | Side effect — no dependency array needed |
| `$props()` | Function props | Component inputs |
| `$bindable()` | *(no equivalent)* | Two-way bindable prop |

### Svelte 4 vs Svelte 5 — Same Feature, Different Philosophy

```js
// Svelte 4 — magic label syntax
let query = '';
$: filtered = countries.filter(c =>
  c.name.common.toLowerCase().includes(query.toLowerCase())
)
```

```js
// Svelte 5 — explicit runes
let query = $state('')
let filtered = $derived(
  countries.filter(c =>
    c.name.common.toLowerCase().includes(query.toLowerCase())
  )
)
```

Both produce identical browser output. The `$:` label is valid JS syntax hijacked by the Svelte compiler — it confuses TypeScript, breaks editor autocomplete, and can't leave `.svelte` files. `$derived` is explicit, works in plain `.js` files, and behaves like any other function call.

### No Dependency Arrays

React requires manually listing what a hook depends on — get it wrong and you get stale state or infinite loops:

```js
// React
useEffect(() => {
  localStorage.setItem('favourites', JSON.stringify([...favourites]))
}, [favourites]) // forget this = bug. Wrong value here = bug.
```

Svelte tracks dependencies automatically at compile time:

```js
// Svelte 5
$effect(() => {
  localStorage.setItem('fav-countries', JSON.stringify([...favourites]))
})
// Svelte sees favourites is read inside — tracks it automatically. No array.
```

### Shared State Without External Libraries

React needs Zustand, Redux, Jotai, or similar for shared cross-component state. In Svelte 5, `$state` works in plain `.js` files — no library needed:

```js
// store.js — plain JS file, no imports
export const appState = $state({ query: '', favourites: new Set() })
```

```svelte
<!-- any component -->
<script>
  import { appState } from './store.js'
</script>
<input bind:value={appState.query} />
```

This was impossible in Svelte 4 — reactive state was trapped inside components. Runes solved the escape problem.

---

## Quadrant 3 — HOW *(Application)*
> *How does it work in practice? The live demo.*

### The App — Europe Explorer

Built with Svelte 5 + Vite. Fetches all European countries from the REST Countries API and demonstrates every major rune in a real context.

| Feature | Rune used | What to show |
|---|---|---|
| Search input filtering | `$derived` | Type → list updates instantly, no event handler |
| Subregion dropdown filter | `$derived` | Combine two reactive sources, zero extra code |
| Favourites (★) | `$state` + `$effect` | State mutates directly; `$effect` persists to localStorage |
| Document title on country open | `$effect` | Side effect triggered by state change, no boilerplate |
| Border country chips | `$derived` | Computed from loaded data, clickable navigation |
| Skeleton loading cards | `$state` (loading flag) | Clean conditional rendering |

**Run it:**

```bash
cd demo
npm install
npm run dev
```

**Demo script:**
1. Open the app — show skeleton loading, then the grid
2. Type in search — `filtered` is `$derived`, updates live
3. Star a country — `$effect` fires, check localStorage in DevTools
4. Click a country — document title changes (`$effect`), border chips load
5. Click a border country chip — navigates via `$state` mutation only, no router

---

## Quadrant 4 — WHAT IF *(Implications)*
> *What does this mean beyond the project? Open questions.*

### Signals Are Winning

Svelte 5 runes are signals — a reactive primitive that tracks its own dependencies. Svelte didn't invent this; it joined a movement already underway:

- **Solid.js** — built on signals from day one
- **Vue 3** — `ref()` and `reactive()` are signals
- **Angular 18+** — added signals as core primitive, deprecating Zone.js
- **Preact** — `@preact/signals` library
- **React** — the last major holdout on VDOM diffing and manual dependency arrays

### What If React Is the Outlier?

For a decade, React's model was the default assumption. The question is now flipping: why does React still require dependency arrays in 2025? Why is state mutation forbidden? The rest of the ecosystem moved on.

### What If DX Is the Only Battleground Left?

Given that runtime performance differences are negligible on modern hardware, and bundle sizes are marginal with compression and caching — the only meaningful competition left between frameworks is: *which one is less painful to write?*

Runes are Svelte's answer. The rest of the industry is converging on the same answer through signals.

**The browser never cared. It was always about the developer.**

---

## Resources

- [Svelte 5 Docs](https://svelte.dev/docs)
- [Runes announcement blog post](https://svelte.dev/blog/runes) — explains the WHY from the creator
- [Rich Harris — Rethinking Reactivity](https://www.youtube.com/watch?v=AdNJ3fydeao) — the philosophy behind the compiler approach
- [REST Countries API](https://restcountries.com)
