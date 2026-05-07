# Svelte 4 vs Svelte 5 Runes — Research Brief

## The Big Picture

Svelte is a frontend framework like React or Vue, but fundamentally different: it's a **compiler**, not a runtime library. Your Svelte code gets compiled to vanilla JS at build time. No virtual DOM, no diffing — the browser gets plain DOM manipulation code. React ships ~45kb of runtime to the browser. Svelte ships near zero.

## The Story (Why This Project Matters)

Svelte 4 was famous for "magic" reactivity:

```js
// Svelte 4 — just write normal JS, compiler makes it reactive
let count = 0;
$: doubled = count * 2; // reactive declaration via label syntax
```

No imports. No hooks. Just write a variable, it's reactive. Devs loved it.

But the magic had real problems:
- Reactive state couldn't leave a component without workarounds
- The `$:` label syntax confused TypeScript and editors
- Hard to reason about what triggered what
- Couldn't pass reactive state into plain `.js` utility files

So Svelte 5 introduced **runes** — explicit reactive primitives:

```js
// Svelte 5 — explicit, but works anywhere
let count = $state(0);
let doubled = $derived(count * 2);
$effect(() => console.log(count));
```

Same power. No magic. Reactive state now works outside components, in plain JS files, passed into functions.

The Svelte team essentially said: **"our best feature was a mistake, here's the better version."**

That tension — implicit magic vs explicit control — is your research question.

---

## The 5 Runes

| Rune | React Equivalent | What it does |
|------|-----------------|--------------|
| `$state()` | `useState()` | reactive variable |
| `$derived()` | `useMemo()` | computed value, auto-updates |
| `$effect()` | `useEffect()` | side effect when deps change |
| `$props()` | function params / props | component inputs |
| `$bindable()` | no direct equivalent | two-way bindable prop |

Key differences from React:
- No dependency arrays — Svelte tracks deps automatically
- No setter function — mutate `$state` directly (`count++` not `setCount(c => c+1)`)
- `$derived` is truly computed — can't manually set it

---

## Research Angles

### 1. Developer Experience
- Which is easier to learn for someone who knows React?
- Where does Svelte 4 magic confuse? Where do runes add clarity?
- Lines of code comparison for same feature

### 2. Architectural Improvement
- What actual problems did runes solve? (shared state, TS support, store complexity)
- Svelte 4 stores (`writable`, `readable`, `derived`) vs runes for shared state

### 3. The Bigger Trend
- Signals are taking over: Angular 18+, Preact Signals, Solid.js, Vue 3 reactivity
- Svelte 5 runes ARE signals — Svelte joined the movement
- React is the last major holdout on VDOM diffing

---

## Resources to Start

- Svelte 5 docs: svelte.dev/docs (has migration guide from v4)
- Svelte 4 REPL: svelte.dev/repl (can write Svelte 4 here)
- Svelte 5 blog post: "Runes" — the official announcement explains the WHY
- Rich Harris (Svelte creator) talks on YouTube — good for understanding the philosophy

## Setup When Ready

```bash
# Svelte 5 (latest)
npm create svelte@latest my-app

# Choose: SvelteKit, TypeScript optional, no extra libs to start
```
