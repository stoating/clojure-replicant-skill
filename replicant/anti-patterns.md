# Replicant Anti-Patterns

## Contents
- State management anti-patterns
- Hiccup anti-patterns
- Event handler anti-patterns
- Life-cycle hook anti-patterns
- Alias anti-patterns
- Performance anti-patterns
- Quick diagnostic checklist

---

## State Management Anti-Patterns

### Putting local state in the component/alias

```clojure
;; WRONG — defalias functions run on every render; local atoms are recreated
(defalias counter [_ _]
  (let [count (atom 0)]   ; ← this is reset every render
    [:div
     [:p @count]
     [:button {:on {:click #(swap! count inc)}} "+"]]))
```

Replicant has no component lifecycle. Local state is not possible. All state belongs in one place (your app atom / state container):

```clojure
;; RIGHT
(defalias counter [{:keys [count on-increment]} _]
  [:div
   [:p count]
   [:button {:on {:click on-increment}} "+"]])

;; Use it:
[myapp.ui/counter {:count (:counter @state)
                   :on-increment [:counter/increment]}]
```

---

### Dereferencing atoms inside hiccup

```clojure
;; WRONG — hiccup is data; derefs here run on every call and bypass Replicant's model
(defn ui []
  [:div @my-atom])   ; ← side-effectful, bypasses state tracking
```

Dereference your atom **before** calling `render`, then pass derived values as arguments:

```clojure
;; RIGHT
(defn ui [state]
  [:div (:value state)])

(add-watch my-atom ::render
  (fn [_ _ _ new-state]
    (r/render el (ui new-state))))
```

---

## Hiccup Anti-Patterns

### Conditionally including the attribute map

```clojure
;; WRONG — the attribute map position must be consistent
[:div
 (when some-condition? {:class "active"})
 "Content"]
```

Replicant (and clj-kondo in dev mode) will warn about this. The issue is that if the condition is false, `"Content"` slides into the attribute position. Always use a stable attr map:

```clojure
;; RIGHT
[:div {:class (when some-condition? "active")}
 "Content"]
```

### Missing `:replicant/key` in dynamic lists

```clojure
;; WRONG — without keys, reordering causes incorrect in-place mutations
[:ul
 (for [{:keys [id name]} items]
   [:li name])]
```

```clojure
;; RIGHT
[:ul
 (for [{:keys [id name]} items]
   [:li {:replicant/key id} name])]
```

Keys must be **unique within their parent** and **stable across renders** (use entity IDs, not indices).

### Using index as `:replicant/key`

```clojure
;; WRONG — index-based keys mean reordering still causes full re-renders
(map-indexed (fn [i item]
               [:li {:replicant/key i} (:name item)])
             items)
```

Use a stable, unique identifier from the data instead.

---

## Event Handler Anti-Patterns

### Putting application logic in event handler functions

```clojure
;; WRONG — logic inside hiccup, untestable without DOM
[:button {:on {:click (fn [_]
                        (swap! state update :count inc)
                        (api/log-click!))}}
 "Click"]
```

Keep hiccup as pure data. Use data-driven dispatch:

```clojure
;; RIGHT
[:button {:on {:click [[:counter/increment] [:analytics/click]]}}
 "Click"]

;; And handle in dispatch fn
```

### Forgetting to register a dispatch function

If you use data-driven handlers but don't call `set-dispatch!`, events are silently dropped. Always register your dispatch function at app startup, before the first render.

---

## Life-cycle Hook Anti-Patterns

### Triggering re-renders inside hooks

```clojure
;; WRONG — calling render inside on-mount causes nested render
[:div {:replicant/on-mount
       (fn [_]
         (r/render el (ui @state)))}]  ; ← Replicant warns about nested renders
```

If you need to react to mount, schedule it on the next tick, or use your state atom's watch mechanism.

### Relying on unmount hook for critical cleanup without transitions

`:replicant/on-unmount` fires before the element is removed. But if you have `:replicant/unmounting` attrs, the hook fires **after** the transition completes (when the element is actually removed). Don't assume immediate cleanup.

### Storing DOM references in app state

```clojure
;; WRONG — DOM nodes do not serialize, are not Clojure values
(swap! state assoc :chart-node (js/document.getElementById "chart"))
```

Use node memory (`remember` / `recall`) for DOM references and third-party JS objects:

```clojure
;; RIGHT
{:replicant/on-mount (fn [{:replicant/keys [node remember]}]
                       (remember {:chart (js/Chart. node #js {})}))}
```

---

## Alias Anti-Patterns

### Registering aliases after first render

Aliases are resolved at render time. If you call `register!` after `render`, the first render won't include your alias. Register all aliases before the first `render` call.

### Using non-namespaced keywords as alias tags

```clojure
;; WRONG — non-namespaced keywords are treated as HTML tags, not aliases
[:button {:on {:click ...}} "Click"]   ; fine — real HTML element
[:ui-button {:on {:click ...}} "Click"] ; also treated as HTML, not alias
```

Aliases require **qualified** keywords:

```clojure
;; RIGHT
[:ui/button {:on {:click ...}} "Click"]
```

### Forgetting that alias functions always receive two positional args

```clojure
;; WRONG — if alias-data is not used, children end up in wrong position
(defalias panel [attrs children]
  ...)
;; When called with alias-data: panel receives (attrs alias-data child1 child2...)
;; "children" only gets alias-data, not the actual children
```

When you pass `:alias-data` to `render`, the alias function signature is `[attr-map alias-data & children]`. If you never use `:alias-data`, declare it as `_`:

```clojure
(defalias panel [attrs _ & children]  ; correct if you don't use alias-data
  ...)
```

---

## Performance Anti-Patterns

### Generating hiccup in event handlers

Don't call your render function from within a dispatch handler and immediately pass the result to `render` — you bypass the natural re-render loop. Let state changes trigger re-renders via a watch:

```clojure
;; WRONG
(r/set-dispatch! (fn [_ [action]]
                   (swap! state handle action)
                   (r/render el (ui @state))))  ; ← double render possible

;; RIGHT — watch triggers render after state change
(add-watch state ::render
  (fn [_ _ _ new-state]
    (r/render el (ui new-state))))
```

### Very deep hiccup trees with no keys

Without keys, Replicant has no hint about identity across renders. In large lists, it may update every element in place even when only one changed. Add `:replicant/key` to list items.

---

## Quick Diagnostic Checklist

| Symptom | Likely cause |
|---------|-------------|
| Event handler fires but nothing updates | Not calling `render` after state change; watch not set up |
| Data-driven handler does nothing | `set-dispatch!` never called |
| Life-cycle hook fires on every render | No `:replicant/key`; Replicant sees a new element each time |
| Mount animation doesn't work | `:replicant/mounting` attr doesn't actually differ from normal attrs |
| Unmount animation gets stuck | Element has `transition-duration` CSS but no transition fires (e.g. height from `auto` → 0) |
| Alias not expanding | Alias not registered before render; using non-namespaced keyword |
| `remember` memory is nil on update | `:replicant/key` changed between renders; Replicant treats it as a new node |
| clj-kondo warns on `defalias` | clj-kondo exports not imported; run `clj-kondo --copy-configs --dependencies --lint ...` |
