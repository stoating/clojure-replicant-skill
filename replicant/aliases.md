# Replicant Aliases

## Contents
- What aliases are
- Defining aliases with `defalias`
- Using `aliasfn` for anonymous alias functions
- Registering aliases with `register!`
- Invoking aliases in hiccup
- Passing data to aliases
- Expanding aliases without rendering
- clj-kondo support

---

## What Aliases Are

Aliases are reusable hiccup fragments identified by a **namespaced keyword**. They are functions that receive an attribute map and children, and return hiccup. Unlike React components, they have no local state and no lifecycle — they are just functions that produce data.

```clojure
;; In hiccup, a namespaced keyword as the tag = alias invocation
[:ui/button {:primary? true} "Log in"]
```

Replicant calls the function registered under `:ui/button` with the attribute map and children, then renders the returned hiccup.

---

## Defining Aliases with `defalias`

`defalias` creates a function, registers it globally, and `def`s the alias keyword:

```clojure
(ns myapp.ui
  (:require [replicant.alias :refer [defalias]]))

(defalias button [{:keys [primary? disabled?]} & children]
  [:button.btn
   {:class (when primary? "btn-primary")
    :disabled disabled?}
   children])
```

The `def`'d var holds the **keyword** `:myapp.ui/button`, not the function:

```clojure
myapp.ui/button  ;=> :myapp.ui/button
```

Use it in hiccup:

```clojure
[myapp.ui/button {:primary? true} "Submit"]
;; equivalent to
[:myapp.ui/button {:primary? true} "Submit"]
```

The alias function signature is `[attr-map & children]`. Both arguments are always passed; unused arguments should be destructured as `_`:

```clojure
(defalias divider [_ _]
  [:hr.divider])
```

---

## Using `aliasfn` for Anonymous Alias Functions

`aliasfn` creates the function without `def`-ing it or registering it globally. Use it when you want to pass alias functions as values:

```clojure
(require '[replicant.alias :refer [aliasfn]])

(def my-aliases
  {:ui/badge
   (aliasfn :ui/badge [{:keys [color]} text]
     [:span.badge {:class (str "badge-" (name color))} text])})
```

`aliasfn` wraps the function with debugging metadata when assertions are enabled, and strips it in production — same as `defalias` but without side effects.

---

## Registering Aliases with `register!`

To register an alias at runtime without `defalias`:

```clojure
(require '[replicant.alias :as alias])

(alias/register! :ui/card my-card-fn)
```

To retrieve all globally registered aliases:

```clojure
(alias/get-registered-aliases)
;; => {:ui/card #object[...], :ui/button #object[...]}
```

---

## Invoking Aliases in Hiccup

Aliases are invoked exactly like normal elements, using namespaced keywords as the tag:

```clojure
;; No children
[:ui/spinner]

;; With attrs only
[:ui/spinner {:size :lg}]

;; With attrs and children
[:ui/card {:title "Welcome"} [:p "Hello!"]]

;; CSS shorthand works too
[:ui/button.btn-lg {:primary? true} "Save"]
;; → called as (button-fn {:class #{"btn-lg"} :primary? true} ...)
```

Classes from the tag shorthand (`:ui/button.btn-lg`) are merged into the attribute map's `:class` key before the function is called.

---

## Passing Data to Aliases

### Via the attribute map

The most common approach — pass everything in the attr map:

```clojure
[:ui/user-card {:user user :show-avatar? true}]
```

### Via `:alias-data` (global context)

Pass shared data (e.g. i18n dictionary, theme) to **all** aliases at render time via `:alias-data`:

```clojure
(r/render el hiccup {:aliases aliases :alias-data {:locale :en :theme :dark}})
```

Inside the alias function, `:alias-data` is the second argument:

```clojure
(defalias greeting [{:keys [name]} {:keys [locale]}]
  [:p (case locale
        :en (str "Hello, " name)
        :nb (str "Hei, " name))])
```

Note: with `alias-data`, the function receives `[attr-map alias-data & children]` — the second positional argument is `:alias-data`, not children.

---

## Expanding Aliases Without Rendering

Use `expand-1` and `expand` for testing or transforming hiccup without rendering to DOM/string:

```clojure
(require '[replicant.alias :as alias])

;; Expand one level of aliases
(alias/expand-1 [:ui/panel {:title "Hi"}]
                {:aliases my-aliases})

;; Expand all levels recursively
(alias/expand [:ui/panel {:title "Hi"}]
              {:aliases my-aliases})
```

These are useful in unit tests to verify alias output without a DOM environment:

```clojure
(deftest button-test
  (is (= (alias/expand-1 [:ui/button {:primary? true} "Submit"]
                          {:aliases aliases})
         [:button.btn.btn-primary {} "Submit"])))
```

---

## clj-kondo Support

Replicant ships clj-kondo exports so `defalias` calls are linted correctly. To activate in your project:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clojure -Spath)"
```

This copies Replicant's clj-kondo config into `.clj-kondo/`. After this, clj-kondo understands `defalias` arity and naming.
