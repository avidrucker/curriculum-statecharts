# Executive Summary — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

The 5-minute refresher. Use this when you've completed ex03c once and want to re-anchor.

---

## The one big idea

**The `convenience` namespace is sugar — every helper emits standard elements.** `on` produces a `transition`. `handle` produces a targetless `transition + script`. `choice` produces a *state* containing eventless guarded transitions.

The most consequential new idea: **`choice` is a transient state.** The chart enters it and immediately leaves via one of its eventless transitions — but if all the guards fail and there's no `:else`, the chart gets STUCK in the choice state. This is the canonical ex03c footgun.

---

## What we built

ex03's login gate, rewritten with convenience helpers. Same behavior, more concise.

```clojure
(def login-with-convenience
  (statechart {}
    (state {:id :idle}
      (handle :enter-password
        (fn [_ data]
          [(ops/assign :password (-> data :_event :data :password))]))
      (on :submit :checking))

    (state {:id :checking}
      (choice {:id :auth-check}
        (fn [_ data] (= (:password data) "secret")) :authenticated
        :else :rejected))

    (state {:id :authenticated})
    (state {:id :rejected}
      (on :retry :idle))))
```

Compared to ex03: the chart is split into entry-data-handler (`handle`), simple transitions (`on`), and the conditional decision (`choice`). Each helper has a different role.

---

## Mechanics in 30 seconds

- **`(on :event :target)`** = `(transition {:event :event :target :target})`. Pure sugar.
- **`(handle :event expr)`** = `(transition {:event :event} (script {:expr expr}))`. Targetless transition + script.
- **`(choice {:id :x} pred-1? :target-1 pred-2? :target-2 :else :fallback)`** = a `(state {:id :x :diagram/prototype :choice})` containing eventless `(transition {:cond ... :target ...})` children plus a final `(transition {:target :fallback})` for `:else`.
- **Eventless transitions** (no `:event` key) fire automatically on state entry. This is what makes `choice` dispatch immediately.
- **Choice is a transient state.** Normally `(t/in? env :choice-id)` returns `false` — the chart passes through within one processing cycle.
- **`:else` is appended last** by `choice`'s code regardless of where you put it in the call. The verbose form has no such forgiveness; document order matters.
- **The convenience namespace is ALPHA, NOT API STABLE** (per its own docstring). The underlying `transition + script + state` patterns from the elements namespace are stable.

---

## Composes with

- **ex01** through **ex03b** are still the foundation; `convenience` adds nothing the elements namespace can't do.
- **ex04 (parallel)** uses `state` directly — no convenience helpers required (though `on` works fine for transitions inside parallel regions).
- **ex06 (history)** doesn't use convenience helpers; history needs the explicit `history` element.
- **ex09 (invocations)** uses `handle`-like patterns extensively in real Fulcro chart code.

---

## Gotchas to remember

- **Choice without `:else` + all-falsy guards = chart hangs.** `(t/in? env :choice-id)` returns `true` (the chart is genuinely stuck there). Always include `:else` unless you can prove every reachable data-model state is covered.
- **Choice's `:id` IS a real state ID.** Don't collide with other state IDs in the chart.
- **`(t/in? env :choice-id)` is *almost always* false** — choice states are transient. The exception is the "stuck" failure mode above.
- **`:_event` is `nil` in choice predicates when entered automatically** (e.g., as the chart's initial state). When entered via a triggering event, `:_event` carries that event.
- **The convenience namespace is ALPHA.** `handle`, `choice`, `on` may rename, change arity, or behave differently in future versions. The elements-namespace forms are the stability anchor for long-lived production code.
- **`choice` is a function, not a macro.** You can `apply` it. There's a separate `convenience-macros/choice` that IS a macro (different namespace) — don't confuse them.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex03c-convenience :as ex03c])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex03c/login-with-convenience
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/in? env :idle)                                                 ; → true

(t/run-events! env {:name :enter-password :data {:password "secret"}})
(:password (t/data env))                                          ; → "secret"

(t/run-events! env :submit)
(t/in? env :authenticated)                                        ; → true
(t/in? env :auth-check)                                           ; → false (transient!)
```

If those checks pass, ex03c is solid. Move on to [ex04 (parallel regions)](../ex04-parallel/) — the first dense module, and where the configuration set finally contains multiple atomic states simultaneously.
