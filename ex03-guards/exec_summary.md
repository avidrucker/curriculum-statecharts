# Executive Summary — ex03: Guards (Conditions) & Document Order

The 5-minute refresher. Use this when you've completed ex03 once and want to re-anchor.

---

## The one big idea

**A `:cond` on a transition is a `(fn [env data] -> truthy)` that decides whether the transition fires.** When multiple transitions on the same state match the same event, the runtime evaluates them in **document order** and fires the *first* whose guard passes. The fallback idiom: guarded-specific-transition first, unguarded-catch-all-transition second.

This is the first exercise where document order is load-bearing — and the introduction of the data model (`session data`) the chart can read/write.

---

## What we built

A login gate. `:submit` has two transitions; the first checks a password; the second is the unconditional fallback.

```clojure
(def login-gate
  (statechart {}
    (state {:id :login}
      (transition {:event  :submit
                   :cond   (fn [_env data] (= (:password data) "secret"))
                   :target :authenticated})
      (transition {:event :submit :target :rejected}))               ; no :cond = fallback
    (state {:id :authenticated})
    (state {:id :rejected}
      (transition {:event :retry :target :login}))))
```

The test seeds `:password` via `t/goto-configuration!` (a test-only helper), then fires `:submit` and asserts the correct branch was taken.

---

## Mechanics in 30 seconds

- **Guard signature: `(fn [env data] -> truthy)`** — env first, data second. Not boolean-strict (`"false"` is truthy because non-`false` non-`nil` is truthy in Clojure).
- **First match wins.** Transitions tried in document order; first one whose `:event` matches AND `:cond` passes (or is absent) fires.
- **A failing guard isn't an error.** The transition is skipped; the ancestor walk from ex02 continues.
- **Guards on inactive states aren't called.** Performance + sanity.
- **No `:cond` means unconditionally enabled.** That's how the fallback pattern works.
- **`data` includes `:_event`.** The current event is at `(get data :_event)`. The event's payload is at `(get-in data [:_event :data])`.

---

## Session data — first encounter

Every running chart has a **data model** — a per-session storage area readable from guards / actions. Before anything writes to it, it's `nil`. After `(t/goto-configuration! env [(op/assign :password "secret")] #{:login})` (test-only) or `(op/assign :password "x")` from an on-entry action (production), it's a map.

`op/assign` returns a **descriptor map** — `{:op :assign :data {:password "x"}}` — not a side-effecting function. The runtime interprets it.

---

## Composes with

- **ex02** established the ancestor walk; ex03 lets a failed guard pass the search up that chain.
- **ex03b** uses the data model in earnest (writing values from on-entry / handle).
- **ex03c** adds `choice` — sugar for "guarded eventless transitions in a transient state" (think `cond` in chart form).

---

## Gotchas to remember

- **A guard that throws is silently caught and treated as `false`.** The chart keeps moving (falls through to the next matching transition). DEBUG logs record the exception but tests pass for the wrong reason.
- **Wrong arity is a runtime error.** `(fn [data] ...)` throws when the runtime calls it with 2 args.
- **`:_event` is in `data`, not `env`.** The mnemonic: `env` is plumbing, `data` is content.
- **`t/goto-configuration!`'s configuration includes `:ROOT`** — unlike the normal `start!` lifecycle, which excludes it.
- **`op/assign` is a descriptor.** Calling it doesn't mutate the data model; it produces a value that a sink (on-entry, goto-configuration!, script) interprets.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex03-guards :as ex03])
(require '[com.fulcrologic.statecharts.testing :as t])
(require '[com.fulcrologic.statecharts.data-model.operations :as op])

;; Correct password
(def env (t/new-testing-env {:statechart ex03/login-gate
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/goto-configuration! env [(op/assign :password "secret")] #{:login})
(t/run-events! env :submit)
(t/in? env :authenticated)        ; → true

;; Wrong password (fresh env)
(def env2 (t/new-testing-env {:statechart ex03/login-gate
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(t/goto-configuration! env2 [(op/assign :password "wrong")] #{:login})
(t/run-events! env2 :submit)
(t/in? env2 :rejected)            ; → true
```

If both pass, ex03 is solid. Move on to [ex03b (data operations)](../ex03b-data-operations/) for `on-entry`, `script`, and `handle` — the chart-side data model.
