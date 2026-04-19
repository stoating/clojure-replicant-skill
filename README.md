# Clojure Replicant Skill

A structured Markdown skill that teaches AI coding agents how to work with Replicant — a data-driven rendering library for Clojure(Script). Covers hiccup syntax, DOM and string rendering, aliases (`defalias`/`aliasfn`), data-driven event handlers, life-cycle hooks (on-mount, on-update, on-unmount), CSS mounting/unmounting transitions, node memory, and the dispatch pattern.

Built in [Claude Code's Agent Skill format](https://github.com/anthropics/skills), but usable with **any agent** that can load Markdown as context (Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, and others via the [agents.md](https://agents.md/) convention).

---

## Installation

Pick the section matching your agent.

### A) Claude Code (recommended — via plugin marketplace)

```
/plugin marketplace add stoating/clojure-replicant-skill
/plugin install replicant@clojure-replicant-skill
```

Once installed, invoke it with:
```
/replicant
```

To update later:
```
/plugin marketplace update clojure-replicant-skill
```

### B) Claude Code (via the stoating marketplace)

```
/plugin marketplace add stoating/plugins
/plugin install replicant@stoating
```

### C) Claude Code (manual copy)

```bash
git clone https://github.com/stoating/clojure-replicant-skill.git
cp -r clojure-replicant-skill/replicant ~/.claude/skills/
```

### D) Cursor, Codex CLI, Aider, Gemini CLI, Windsurf, Cline, Zed, Amp

All of these honor `AGENTS.md`. Drop this repo (or just `AGENTS.md` + `replicant/`) at your project root and the agent will read it on session start.

### E) Any other agent — generic fallback

Attach or paste the contents of `replicant/SKILL.md` as system instructions or context, then attach individual reference files as needed.

---

## Repository layout

```
.
├── .claude-plugin/
│   ├── marketplace.json       # Claude Code marketplace manifest
│   └── plugin.json            # Claude Code plugin manifest
├── AGENTS.md                  # Cross-agent entry point (agents.md convention)
├── README.md                  # This file
├── LICENSE
└── replicant/                 # The actual skill
    ├── SKILL.md               # Entry point — decision table
    ├── core-concepts.md       # Hiccup syntax, render, unmount, set-dispatch!, SSR, install
    ├── aliases.md             # defalias, aliasfn, register!, expand, alias-data
    ├── event-handlers.md      # Function and data-driven handlers, dispatch, interpolation
    ├── life-cycle-hooks.md    # on-mount/update/unmount, replicant/key, transitions, memory
    └── anti-patterns.md       # Common mistakes and how to fix them
```

---

## What the skill covers

| File | Contents |
|---|---|
| `SKILL.md` | Index, decision table — the agent always starts here |
| `core-concepts.md` | Design philosophy, hiccup syntax, `replicant.dom/render`, `replicant.string/render`, `set-dispatch!`, installation, build options |
| `aliases.md` | `defalias`, `aliasfn`, `register!`, `get-registered-aliases`, `expand`/`expand-1`, alias-data, clj-kondo |
| `event-handlers.md` | Function handlers, data-driven handlers, dispatch function, event data interpolation, handler options (capture/passive/once) |
| `life-cycle-hooks.md` | Mount/update/unmount hooks, context map, `replicant/key`, CSS transitions (mounting/unmounting), node memory (`remember`/`recall`) |
| `anti-patterns.md` | State in aliases, missing keys, no dispatch function, incorrect hiccup structure, performance issues |

---

## Sources

- [Replicant source](https://github.com/cjohansen/replicant) — canonical source and tests
- [Replicant user guide](https://replicant.fun/learn/) — official documentation
- [Reference API docs](https://cljdoc.org/d/no.cjohansen/replicant/) — cljdoc API reference

**Skill-authoring references:**
- [Anthropic Agent Skills — Best Practices](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [anthropics/skills](https://github.com/anthropics/skills) — canonical examples and spec
- [agents.md](https://agents.md/) — cross-agent `AGENTS.md` convention
- [Claude Code plugin-marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces)

---

## License

See [LICENSE](LICENSE).

This skill is a curated digest of the Replicant source code and documentation, intended to give an AI coding agent accurate, actionable knowledge of Replicant without requiring it to read and parse the raw source on every task.
