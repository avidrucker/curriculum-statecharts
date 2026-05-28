# Executive Summary — ex03b: Data Model Operations & Event Data

The 5-minute refresher. Use this when you've completed ex03b once and want to re-anchor.

---

## The one big idea

**A `script`'s `:expr` returns a *vector of operation descriptors*, not a single op.** This vector-wrapping requirement is the single most common bug when writing data-modifying chart elements. Forgetting the vector silently does nothing.

ex03b also introduces three composable mechanics: **`on-entry`** for state-entry actions, **`handle`** (convenience) for targetless event handlers that update data without changing state, and **`(-> data :_event :data :key)`** for reading event payload from inside expressions.

---

## What we built

A counter. One state `:active`. Initialize `:count` to 0 on entry. Handle `:increment`, `:decrement` (floor at 0), `:set-value` (reads event payload), `:reset`. Every event updates `:count` without changing state.

```clojure
(def cart-chart
  (statechart {}
    (state {:id :active}
      (on-entry {} (script {:expr (fn [_ _] [(ops/assign :count 0)])}))

      (handle :increment
        (fn [_ data] [(ops/assign :count (inc (:count data)))]))
      (handle :decrement
        (fn [_ data] [(ops/assign :count (max 0 (dec (:count data))))]))
      (handle :set-value
        (fn [_ data]
          [(ops/assign :count (or (-> data :_event :data :value) 0))]))
      (handle :reset
        (fn [_ _] [(ops/assign :count 0)])))))
```

(The variable name `cart-chart` is a leftover from an earlier exercise version. The topic is a counter.)

---

## Mechanics in 30 seconds

- **`(on-entry {} <executable-content>)`** — runs the content every time the state is entered (not just on first entry).
- **`(script {:expr f})`** — wraps an expression that returns a vector of ops.
- **The vector-of-ops contract:** `:expr` MUST return `[op1 op2 ...]`. Single op `(ops/assign :x 1)` is silently ignored. `nil` and `[]` are valid no-ops.
- **`handle` (convenience)** = sugar for `(transition {:event :e} (script {:expr f}))` — a targetless transition with a script body. No state change; just side effects.
- **Targetless transition** = `:target` is `nil` or absent. The transition fires, runs its body, and the chart stays in the same configuration.
- **`(t/data env)`** — test helper. Returns the data model (nil before `start!`, populated after).
- **Event payload via `:_event`:** for `(t/run-events! env {:name :foo :data {:k v}})`, the expression reads `(-> data :_event :data :k)`.

---

## Composes with

- **ex03** introduced `:cond` guards (same `(fn [env data])` signature; different return type — boolean vs op vector).
- **ex03c** adds `choice` — eventless conditional transitions used as a dispatch mechanism, often combined with `handle`s upstream that put values into the data model.
- **ex07** introduces `on-exit` and `:type :internal` for transitions that should NOT re-fire on-entry/on-exit.
- **ex09** uses `script`/`expr` extensively for invocation parameter passing.

---

## Gotchas to remember

- **Missing vector wrapping** — the canonical ex03b bug. `(fn [_ _] (ops/assign :x 1))` silently does nothing; the data model never updates.
- **`on-entry`'s `data` is `nil` on first entry.** Reading from it with `(:count data)` returns `nil`; `(inc nil)` throws NPE. Pure-write or use `(or v 0)` defaults.
- **`handle` returns `:target nil`** in the emitted transition. That's intentional — targetless.
- **`handle` is from `convenience`** (ALPHA, not API stable). The verbose `(transition + script)` form is safer for long-lived production charts.
- **`(t/data env)` returns `nil` before `(t/start! env)`** — the data model isn't populated until the chart starts.
- **The default data model is flat.** `(ops/assign :user/name "x")` stores at the top level; vector paths don't `assoc-in`.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex03b-data-operations :as ex03b])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex03b/cart-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/data env)                                         ; → {:count 0}
(t/run-events! env :increment :increment :increment)
(t/data env)                                         ; → {:count 3}
(t/run-events! env {:name :set-value :data {:value 42}})
(t/data env)                                         ; → {:count 42}
```

If those checks pass, ex03b is solid. Move on to [ex03c (convenience helpers)](../ex03c-convenience/) for the rest of the convenience namespace (`on`, `choice`).
