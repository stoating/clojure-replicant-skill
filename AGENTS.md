# Agent instructions

This repository contains the **`replicant`** skill — a structured set of Markdown files that teach an AI coding agent how to work with Replicant, the data-driven rendering library for Clojure(Script).

**Any agent that understands this `AGENTS.md` convention should:**

1. Treat `replicant/SKILL.md` as the entry point — it contains a decision table pointing to the right reference file for the task.
2. Load reference files on demand based on that table:
   - `core-concepts.md` — design philosophy, hiccup syntax, `replicant.dom/render`, `replicant.string/render`, `set-dispatch!`, installation
   - `aliases.md` — `defalias`, `aliasfn`, `register!`, `expand`/`expand-1`, passing alias-data, clj-kondo support
   - `event-handlers.md` — function handlers, data-driven handlers, dispatch function, interpolation, handler options
   - `life-cycle-hooks.md` — `on-mount`, `on-update`, `on-unmount`, `replicant/key`, mounting/unmounting transitions, node memory
   - `anti-patterns.md` — common mistakes and things to avoid
3. Never suggest putting local state or application logic inside alias functions — all state belongs outside Replicant.
4. Prefer data-driven event handlers over function handlers — the dispatch pattern keeps hiccup as pure data.
5. Always recommend `:replicant/key` on dynamic list items.

This file follows the [agents.md](https://agents.md/) convention and is honored by OpenAI Codex CLI, Cursor, Aider, Zed, Amp, Gemini CLI, Google Jules, Windsurf, Factory, RooCode, and many others.

For Claude Code, the richer native format is `.claude-plugin/` + `replicant/SKILL.md`.
