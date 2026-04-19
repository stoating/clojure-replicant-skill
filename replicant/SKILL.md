---
name: replicant
description: Use when working with Replicant — a data-driven rendering library for Clojure(Script). Activate when the user asks about rendering hiccup to the DOM or strings, virtual DOM reconciliation, aliases (defalias/aliasfn), data-driven event handlers, life-cycle hooks (on-mount, on-update, on-unmount), mounting/unmounting animations, replicant/key, node memory (remember/recall), replicant.dom/render, replicant.string/render, set-dispatch!, or structuring a pure-function UI with Replicant.
version: 1.0.0
---

# Replicant

Replicant is a data-driven rendering library for Clojure(Script). It renders hiccup to DOM nodes (ClojureScript) or strings (Clojure or ClojureScript). The central idea: **the UI is a pure function of application state**. Call it. Get hiccup. Repeat. Replicant does exactly what's needed to update the DOM.

No components. No local state. No subscriptions. No framework lifecycle. Just data in, hiccup out.

## Quick Decision: What do you need?

| Task | Go to |
|------|-------|
| Understand the rendering model, hiccup syntax, `render`, `unmount`, `set-dispatch!` | [core-concepts.md](core-concepts.md) |
| Define and use reusable hiccup fragments with `defalias` / `aliasfn` | [aliases.md](aliases.md) |
| Wire up click handlers, keyboard events, data-driven dispatch | [event-handlers.md](event-handlers.md) |
| Mount/unmount callbacks, CSS transition hooks, `replicant/key`, node memory | [life-cycle-hooks.md](life-cycle-hooks.md) |
| Server-side rendering to strings, `replicant.string/render` | [core-concepts.md](core-concepts.md) |
| Something isn't working / unexpected behavior / things to avoid | [anti-patterns.md](anti-patterns.md) |

## Core Mental Model

```
app-state → (render-fn app-state) → hiccup → replicant.dom/render → DOM
```

- `render-fn` is a **pure function** — no side effects, no local atoms
- Call `replicant.dom/render` every time state changes; Replicant diffs and patches the DOM
- Event handlers and life-cycle hooks communicate back to you via a **dispatch function** you register with `set-dispatch!`
- Reusable hiccup fragments are **aliases** — functions registered under a namespaced keyword
- Replicant has no opinion on state management; use an atom, a signal library, or anything else
