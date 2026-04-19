# Replicant Core Concepts

## Contents
- Design philosophy
- Hiccup syntax
- DOM rendering (`replicant.dom`)
- String rendering (`replicant.string`)
- Global dispatch
- Dependency and installation
- Build options

---

## Design Philosophy

The user interface is a function of application state:

```clojure
(defn ui [state]
  [:div.app
   [:h1 (:title state)]
   [:p (:body state)]])
```

When state changes, call `ui` again and pass the result to `replicant.dom/render`. Replicant compares the new hiccup to what it rendered last time and makes the minimum necessary DOM mutations. That is the entire programming model.

**What Replicant is not:**
- Not a state management library — bring your own atom/signal
- Not a component framework — there are no components or local state
- Not an async renderer — renders happen synchronously (with transition hooks on next-frame)
- Not a networking layer — use whatever HTTP library you prefer

---

## Hiccup Syntax

Replicant accepts standard Clojure hiccup vectors:

```clojure
[:tag-name optional-attr-map & children]
```

### Tag shorthand

```clojure
:div                   ; plain div
:div#main              ; id="main"
:div.container         ; class="container"
:div#main.container.mt-4  ; id + multiple classes
```

### Attribute map

```clojure
[:input {:type "text"
         :placeholder "Search…"
         :value (:q state)}]
```

Attributes are optional. When omitted, the first non-keyword child is treated as content:

```clojure
[:p "Hello world"]
[:p {:class "lead"} "Hello world"]
```

### Classes

The `:class` attribute accepts a string, keyword, symbol, set, or vector:

```clojure
{:class "btn btn-primary"}
{:class :btn}
{:class #{"btn" "btn-primary"}}
{:class ["btn" (when active? "active")]}   ; nil entries are ignored
```

Classes from the tag shorthand and `:class` are merged.

### Styles

`:style` is a map of CSS property keywords to values:

```clojure
{:style {:color "red"
         :font-size "1rem"
         :margin-top 8}}   ; numeric px values auto-converted to "8px"
```

Some properties (e.g. `:z-index`, `:opacity`) are not pixelized.

### Special attributes

| Attribute | Purpose |
|-----------|---------|
| `:on` | Event handlers map — see [event-handlers.md](event-handlers.md) |
| `:replicant/key` | Stable identity across renders — see [life-cycle-hooks.md](life-cycle-hooks.md) |
| `:replicant/on-mount` | Mount life-cycle hook |
| `:replicant/on-update` | Update life-cycle hook |
| `:replicant/on-unmount` | Unmount life-cycle hook |
| `:replicant/mounting` | Attrs to apply on first frame (for CSS transitions) |
| `:replicant/unmounting` | Attrs to apply while unmounting (for CSS transitions) |
| `:innerHTML` | Set raw HTML string (use sparingly) |
| `:default-value` | Sets the `value` HTML attribute without controlling the input |
| `:default-checked` | Sets the `checked` HTML attribute without controlling the input |
| `:default-selected` | Sets the `selected` HTML attribute without controlling the input |

### Children

Children can be strings, numbers, other hiccup vectors, or sequences/lists of the above. `nil` children are silently skipped:

```clojure
[:ul
 (for [item items]
   [:li {:replicant/key (:id item)} (:name item)])
 (when show-empty? [:li.empty "No items"])]
```

A top-level list or seq of hiccup nodes is also valid:

```clojure
(replicant.dom/render el (list [:h1 "Title"] [:p "Body"]))
```

---

## DOM Rendering (`replicant.dom`)

### `render`

```clojure
(require '[replicant.dom :as r])

(r/render js/document.body hiccup)
(r/render js/document.body hiccup {:aliases aliases :alias-data data})
```

- `el` — any DOM element (the container)
- `hiccup` — a single hiccup vector, or a list/seq of hiccup nodes
- Options map (optional):
  - `:aliases` — map of `qualified-keyword → fn` for custom alias expansion; defaults to globally registered aliases
  - `:alias-data` — extra data passed to alias functions as the second argument

**First call:** clears `el`'s `innerHTML` and renders fresh.  
**Subsequent calls:** diffs against the last rendered hiccup and patches the DOM.

Returns `el`.

### `unmount`

```clojure
(r/unmount js/document.body)
```

Runs unmount hooks, removes all children, and clears internal Replicant state for that element. Safe to call while a render is in progress (deferred to next frame).

### `set-dispatch!`

```clojure
(r/set-dispatch! (fn [event actions] ...))
```

Registers a global function that handles data-driven event handlers and life-cycle hooks. The function receives:
- `event` — map with `:replicant/dom-event` (the browser Event), `:replicant/node` (the DOM element), and `:replicant/trigger` (`:replicant/on-mount`, etc. for hooks)
- `actions` — the data value from the handler, e.g. `[:increment]` or a vector of action tuples

See [event-handlers.md](event-handlers.md) for details.

### `recall`

```clojure
(r/recall some-dom-node)
```

Returns the memory associated with `some-dom-node` outside of an event handler or life-cycle hook. Useful when integrating third-party JS libraries. See [life-cycle-hooks.md](life-cycle-hooks.md).

---

## String Rendering (`replicant.string`)

Works on both the JVM and in ClojureScript. Useful for server-side rendering, email HTML, or tests.

```clojure
(require '[replicant.string :as s])

(s/render [:div.card [:h2 "Hello"] [:p "World"]])
;; => "<div class=\"card\"><h2>Hello</h2><p>World</p></div>"

(s/render hiccup {:aliases aliases :alias-data data})
```

- Self-closing tags (`br`, `img`, `input`, etc.) are rendered correctly.
- `:innerHTML` is rendered as-is (not escaped).
- `:style` and `:class` are serialized properly.
- `:on` event handlers are ignored (no DOM).

`replicant.string/render` returns a string. It does **not** maintain any state between calls.

---

## Global Dispatch

The dispatch pattern decouples event handling from the rendering function:

```clojure
;; 1. Hiccup expresses intent as data
[:button {:on {:click [:user/log-in {:provider :github}]}}
 "Log in with GitHub"]

;; 2. A single dispatch function handles all events
(r/set-dispatch!
  (fn [{:replicant/keys [dom-event]} actions]
    ;; actions = [:user/log-in {:provider :github}]
    (handle-action actions)))
```

For multiple actions per event, use a vector of action tuples:

```clojure
{:on {:click [[:user/log-in {:provider :github}]
              [:analytics/track :login-click]]}}
```

The dispatch function receives `(dispatch-ctx actions)` where `dispatch-ctx` is a map:

| Key | Value |
|-----|-------|
| `:replicant/dom-event` | The browser `Event` object |
| `:replicant/node` | The DOM element that triggered the event |
| `:replicant/trigger` | Hook trigger name for life-cycle hooks (`:replicant/on-mount` etc.) |
| `:replicant/remember` | Function to store memory on the node (in hooks) |
| `:replicant/memory` | Previously stored memory for this node (in hooks) |

---

## Dependency and Installation

Add to `deps.edn`:

```clojure
no.cjohansen/replicant {:mvn/version "2025.12.1"}
```

No transitive dependencies. Zero.

For shadow-cljs (`shadow-cljs.edn` / `package.json`):

```clojure
;; shadow-cljs.edn
{:dependencies [[no.cjohansen/replicant "2025.12.1"]]}
```

---

## Build Options

Control Replicant's behavior in development vs production with compiler constants:

```clojure
;; In your shadow-cljs build config, :closure-defines:
{"replicant.assert/ASSERT" true}   ; enable development assertions (default: false)
```

With assertions enabled, Replicant warns about:
- Conditionally including the attribute map
- Missing aliases
- Nested render calls

**Production builds** should leave assertions disabled (the default) to avoid any overhead.

For ClojureScript, assertions are controlled by the `:optimizations` level — advanced compilation disables `js/goog.DEBUG` automatically.
