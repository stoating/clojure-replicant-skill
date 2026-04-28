# Replicant Life-cycle Hooks

## Contents
- Available hooks
- Hook context map
- `replicant/key` — stable node identity
- Mounting / unmounting CSS transitions
- Node memory (`remember` / `recall`)
- Hooks with data-driven dispatch
- Testing hooks

---

## Available Hooks

Life-cycle hooks are attributes on any hiccup element. They fire when Replicant creates, updates, or removes that DOM node.

| Attribute | When it fires |
|-----------|--------------|
| `:replicant/on-mount` | After the element is inserted into the DOM |
| `:replicant/on-update` | After the element's attributes or children change |
| `:replicant/on-unmount` | Before the element is removed from the DOM |

Values can be a function or a data value (dispatched via `set-dispatch!`):

```clojure
;; Function hook
[:div {:replicant/on-mount (fn [ctx] (js/console.log "mounted" ctx))}
 "Hello"]

;; Data-driven hook
[:div {:replicant/on-mount [:my-hook/mounted]}
 "Hello"]
```

---

## Hook Context Map

Every hook function receives a single map argument with the following keys:

| Key | Available in | Value |
|-----|-------------|-------|
| `:replicant/node` | all hooks | The DOM element |
| `:replicant/trigger` | all hooks | The hook keyword (`:replicant/on-mount`, etc.) |
| `:replicant/remember` | mount, update | Function `(fn [data])` — stores data on the node |
| `:replicant/memory` | update, unmount | Previously stored memory for this node |
| `:replicant/dom-event` | N/A for hooks | Always `nil` for hooks (present only for event handlers) |

```clojure
[:div {:replicant/on-mount
       (fn [{:replicant/keys [node remember]}]
         ;; Integrate a third-party JS library and save a reference
         (let [chart (js/Chart. node #js {})]
           (remember {:chart chart})))

       :replicant/on-update
       (fn [{:replicant/keys [memory]}]
         (let [chart (:chart memory)]
           (.update chart)))

       :replicant/on-unmount
       (fn [{:replicant/keys [memory]}]
         (.destroy (:chart memory)))}]
```

---

## `replicant/key` — Stable Node Identity

By default, Replicant matches elements by position. Add `:replicant/key` to give an element a stable identity that persists across renders regardless of position:

```clojure
[:ul
 (for [{:keys [id name]} items]
   [:li {:replicant/key id} name])]
```

**When to use `:replicant/key`:**
- Lists where items can be reordered, added, or removed
- Elements with `:replicant/on-mount` that should not re-fire when siblings change
- Elements whose life-cycle hooks hold state (memory) that should survive reorder

Without a key, reordering will cause Replicant to update in-place (potentially misattributing memory) rather than moving nodes.

---

## Mounting / Unmounting CSS Transitions

Use `:replicant/mounting` and `:replicant/unmounting` to animate elements in and out with CSS transitions.

### `:replicant/mounting`

The attribute map to apply on the **first render frame**, before the transition-duration elapses. Replicant starts with these attrs, then transitions to the normal attrs on the next frame:

```clojure
[:div {:style {:opacity 1 :transition "opacity 0.3s"}
       :replicant/mounting {:style {:opacity 0}}}
 "Fades in"]
```

On mount: opacity starts at 0 (from `:replicant/mounting`), then Replicant sets opacity to 1 on the next animation frame, triggering the CSS transition.

### `:replicant/unmounting`

Attributes to apply while the element is being removed. Replicant applies these, waits for `transitionend` (or a timeout), then removes the element:

```clojure
[:div {:style {:opacity 1 :transition "opacity 0.3s"}
       :replicant/unmounting {:style {:opacity 0}}}
 "Fades out on removal"]
```

### Combined mount + unmount animation

```clojure
(let [transition-attrs {:style {:background-color "blue"
                                :transition "background-color 0.5s"}
                        :replicant/mounting {:style {:background-color "red"}}
                        :replicant/unmounting {:style {:background-color "red"}}}]
  [:div transition-attrs "Animated"])
```

**Note:** The element must actually change a CSS property that has a `transition-duration` for `transitionend` to fire. If no transition occurs, Replicant falls back to a timer to avoid leaving the element stuck.

---

## Node Memory (`remember` / `recall`)

Node memory lets you associate arbitrary Clojure data with a DOM node — useful for storing references to third-party JS objects without putting them in your app state.

### Storing memory

Call `:replicant/remember` inside a hook:

```clojure
[:canvas {:replicant/on-mount
          (fn [{:replicant/keys [node remember]}]
            (let [ctx (.getContext node "2d")]
              (remember {:ctx ctx})))}]
```

### Accessing memory in subsequent hooks

`:replicant/memory` is available in `:replicant/on-update` and `:replicant/on-unmount`:

```clojure
[:canvas {:replicant/on-update
          (fn [{:replicant/keys [memory]}]
            (draw (:ctx memory)))

          :replicant/on-unmount
          (fn [{:replicant/keys [memory]}]
            ;; clean up
            (.destroy (:ctx memory)))}]
```

### Accessing memory outside hooks

Use `replicant.dom/recall` to retrieve memory from any DOM node outside of a hook:

```clojure
(require '[replicant.dom :as r])

(let [node (js/document.getElementById "my-canvas")
      {:keys [ctx]} (r/recall node)]
  (draw ctx))
```

---

## Hooks with Data-Driven Dispatch

Hooks work with the same dispatch function as event handlers:

```clojure
[:div {:replicant/on-mount [:my-app/element-mounted {:key :chart}]}]
```

In the dispatch function:

```clojure
(r/set-dispatch!
  (fn [{:replicant/keys [node remember trigger]} actions]
    (case (first actions)
      :my-app/element-mounted
      (let [chart (js/Chart. node #js {})]
        (remember {:chart chart}))

      ;; ...
      nil)))
```

Note: `trigger` will be `:replicant/on-mount` for this hook. `dom-event` will be `nil`.

---

## Testing Hooks

In tests, use `replicant.mutation-log` (the test renderer) to verify life-cycle behavior without a real DOM:

```clojure
(require '[replicant.test-helper :as h])

(let [result (h/render [:div {:replicant/on-mount [:my/hook]}])]
  (is (= (h/get-hooks result)
         [{:replicant/trigger :replicant/on-mount
           :actions [:my/hook]}])))
```

See the test files in the replicant source for detailed patterns.
