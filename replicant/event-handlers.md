# Replicant Event Handlers

## Contents
- Function handlers
- Data-driven handlers
- The dispatch function
- Interpolating event data
- Handler options (capture, passive, once)
- Multiple actions per event
- Accessing the DOM event in data handlers

---

## Function Handlers

The simplest form: a plain function in the `:on` map:

```clojure
[:button {:on {:click (fn [e] (js/console.log "clicked" e))}}
 "Click me"]
```

The function receives the browser `Event` object directly. This works but keeps logic inside the hiccup, making testing harder.

---

## Data-Driven Handlers

Data-driven handlers are the preferred pattern. Instead of a function, provide a data value:

```clojure
[:button {:on {:click [:user/log-in {:provider :github}]}}
 "Log in with GitHub"]
```

When Replicant sees a non-function as the event value, it calls the **dispatch function** you registered with `set-dispatch!`, passing a context map and the handler data.

Benefits:
- Hiccup stays pure data — serializable, inspectable, testable
- All application logic lives in one place (the dispatch function)
- Easier to implement undo/redo, logging, time-travel debugging

---

## The Dispatch Function

Register a global dispatch function once at startup:

```clojure
(require '[replicant.dom :as r])

(r/set-dispatch!
  (fn [{:replicant/keys [dom-event node]} actions]
    ;; dispatch-ctx keys:
    ;;   :replicant/dom-event — the browser Event object
    ;;   :replicant/node      — the DOM element
    ;;   :replicant/trigger   — hook trigger (for life-cycle hooks)
    ;;   :replicant/remember  — fn to store node memory (hooks only)
    ;;   :replicant/memory    — previously stored memory (hooks only)
    (handle actions)))
```

The `actions` argument is whatever you put in the `:on` handler — typically a keyword, a vector tuple, or a vector of tuples.

### Typical dispatch pattern

```clojure
(defn handle-action [state [action & args]]
  (case action
    :counter/increment (update state :count inc)
    :counter/set       (assoc state :count (first args))
    :nav/go-to         (assoc state :route (first args))
    state))

(r/set-dispatch!
  (fn [_ actions]
    (swap! app-state handle-action actions)))
```

---

## Multiple Actions per Event

Pass a **vector of action tuples** to fire multiple actions from one event:

```clojure
[:button {:on {:click [[:user/log-in {:provider :github}]
                       [:analytics/track :login-click]]}}
 "Log in"]
```

Replicant calls the dispatch function once per action tuple, in order.

Single-action shorthand (one tuple, not wrapped):

```clojure
{:on {:click [:counter/increment]}}
```

---

## Interpolating Event Data

Often you need values from the DOM event itself (e.g. input value). Interpolate with placeholder keywords that you resolve in the dispatch function:

```clojure
[:input {:type "text"
         :on {:input [:search/query-changed :event.target/value]}}]

;; In dispatch:
(defn interpolate [dom-event args]
  (clojure.walk/postwalk
   (fn [x]
     (case x
       :event.target/value (.. dom-event -target -value)
       :event.target/checked (.. dom-event -target -checked)
       x))
   args))

(r/set-dispatch!
  (fn [{:replicant/keys [dom-event]} [action & args]]
    (let [args (cond->> args dom-event (interpolate dom-event))]
      (swap! app-state handle-action (into [action] args)))))
```

---

## Handler Options

Pass a third element to `{:on {event [handler options]}}` to set `addEventListener` options:

```clojure
;; Capture phase
{:on {:click [[:some/action] {:capture true}]}}

;; Passive (for scroll performance)
{:on {:scroll [[:scroll/track] {:passive true}]}}

;; Fire once
{:on {:click [[:one-time/action] {:once true}]}}
```

The options map is passed directly to `addEventListener` as the options object.

---

## Accessing the DOM Event in Data Handlers

Your dispatch function always receives the `dom-event` via the context map:

```clojure
(r/set-dispatch!
  (fn [{:replicant/keys [dom-event]} actions]
    ;; Prevent default when needed:
    (when (= (first actions) :form/submit)
      (.preventDefault dom-event))
    (handle actions)))
```

For life-cycle hooks, `dom-event` will be `nil` — use `:replicant/trigger` to distinguish:

```clojure
(r/set-dispatch!
  (fn [{:replicant/keys [dom-event trigger]} actions]
    (if trigger
      (handle-hook trigger actions)
      (handle-event dom-event actions))))
```
