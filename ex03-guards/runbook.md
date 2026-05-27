# Runbook — ex03: Guards (Conditions) & Document Order

The operational pass. By the end of this document you'll have edited `src/exercises/ex03_guards.cljc` to define a working login-gate chart and seen all four assertions go green.

**Estimated time:** 25–40 minutes if you've completed ex02. The chart is the same size as ex02's but introduces three new concepts at once (guards, the data model, `goto-configuration!`).

---

## Prerequisites

- You've completed ex01 and ex02 — your familiarity with `state`, `transition`, `t/start!`, `t/in?`, `t/run-events!` is automatic.
- You've read [`tutorial.md`](./tutorial.md) for ex03. The runbook references its terms (guard, fallback, session data, document order tiebreak) without re-defining them.

The cross-cutting [`glossary.md`](../glossary.md) covers anything from earlier exercises you've forgotten.

---

## Where commands run

Same setup as ex01/ex02:

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` — interactive REPL | from the exercise repo root |
| **Shell** | `clojure -M:test` — the kaocha test runner | from the exercise repo root |

All paths below are relative to `statechart-exercises/` (the sibling of this curriculum repo).

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex03_guards.cljc` in your editor. ~62 lines. Three pieces to notice:

1. **The docstring's hint** spells out the guard signature: `(fn [env data] boolean?)`. The `data` map contains the session's local data. Memorize that order — `[env data]`, not `[data env]`. The library will not warn you if you swap them.
2. **The `(:require ...)`** *already* includes `[com.fulcrologic.statecharts.data-model.operations :as op]`. You don't have to add that one. But it does **not** require `statecharts.elements` — same first-move you made in ex01 and ex02.
3. **The `deftest`** is the spec. Four assertions, split across **two** `let` blocks (each block is a fresh env for an independent scenario):
   - Block 1: starts in `:login`; correct password lands in `:authenticated`.
   - Block 2: wrong password lands in `:rejected`; `:retry` from `:rejected` returns to `:login`.

Note the test uses `(t/goto-configuration! env [(op/assign :password "secret")] #{:login})` — that's a *test-time-only* helper that sets the data model and configuration directly. See [tutorial Step 4](./tutorial.md) for why it exists.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start a watch-mode runner:

```sh
clojure -M:test --focus exercises.ex03-guards/guards-test --watch
```

Skip the noisy per-assertion env dumps and DEBUG log output. The summary line is what matters:

```
1 tests, 4 assertions, 4 failures.
```

This time the failure is different from ex01/ex02 in one subtle way: **the chart's `def` itself is `(statechart {})` — an empty chart**. `(t/goto-configuration! env [(op/assign :password "secret")] #{:login})` will *throw* because `:login` isn't a valid state in an empty chart. The first assertion's failure block will show an exception, not just "in? returned false."

Keep the watch terminal open.

---

## Step 3 — Add the elements require

The exercise stub doesn't import `state` or `transition`. Same first move as ex01/ex02:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.data-model.operations :as op]
    [com.fulcrologic.statecharts.elements :refer [state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. The test runner re-runs and still reports `4 failures` — expected.

---

## Step 4 — Build the chart, layer by layer

Five small edits. Each one resolves a specific assertion; the test runner's summary line should drop by one each time.

### 4.1 — The `:login` state alone

Replace `;; YOUR CODE HERE`:

```clojure
(def login-gate
  (statechart {}
    (state {:id :login})))
```

Save.

Expected — **first assertion passes, the rest fail in various ways**:

```
1 tests, 4 assertions, 3 failures.
```

`(t/start! env)` enters the chart; document order picks `:login` as the initial state. `(t/in? env :login)` is now true. The next assertion fails because `goto-configuration! env [...] #{:login}` works (since `:login` exists), then `(t/run-events! env :submit)` does nothing (no transition matches), and `(t/in? env :authenticated)` returns false (no such state).

### 4.2 — Add `:authenticated` and the guarded transition

The second assertion needs:

1. An `:authenticated` state for `:target`.
2. A transition on `:submit` from `:login`, guarded by the password check.

```clojure
(def login-gate
  (statechart {}
    (state {:id :login}
      (transition {:event  :submit
                   :cond   (fn [_env data] (= (:password data) "secret"))
                   :target :authenticated}))
    (state {:id :authenticated})))
```

Save.

Expected — **two assertions pass**:

```
1 tests, 4 assertions, 2 failures.
```

With password `"secret"` in the data model, the guard returns true and the transition fires. The chart enters `:authenticated`. The third assertion (wrong password → `:rejected`) still fails because there's no `:rejected` state and no fallback transition.

### 4.3 — Add the `:rejected` fallback transition + state

The third assertion needs:

1. A `:rejected` state.
2. A second `:submit` transition with no `:cond`, declared **after** the guarded one in document order, targeting `:rejected`.

```clojure
(def login-gate
  (statechart {}
    (state {:id :login}
      (transition {:event  :submit
                   :cond   (fn [_env data] (= (:password data) "secret"))
                   :target :authenticated})
      (transition {:event :submit :target :rejected}))
    (state {:id :authenticated})
    (state {:id :rejected})))
```

Save.

Expected — **three assertions pass**:

```
1 tests, 4 assertions, 1 failures.
```

Now with password `"wrong"`, the guard returns false → runtime moves to the next matching transition → unguarded `:submit` → fires; chart enters `:rejected`.

**This is the moment the fallback pattern earns its keep.** Read your chart and confirm: the guarded transition is *first* in document order, the unguarded fallback is *second*. Swapping them (Try-breaking-it #1 below) is the canonical "I broke my login flow by reordering transitions" bug.

### 4.4 — Add the `:retry` transition on `:rejected`

The fourth assertion needs a way out of `:rejected` back to `:login`:

```clojure
(def login-gate
  (statechart {}
    (state {:id :login}
      (transition {:event  :submit
                   :cond   (fn [_env data] (= (:password data) "secret"))
                   :target :authenticated})
      (transition {:event :submit :target :rejected}))
    (state {:id :authenticated})
    (state {:id :rejected}
      (transition {:event :retry :target :login}))))
```

Save.

Expected — **all four assertions pass**:

```
1 tests, 4 assertions, 0 failures.
```

You've solved ex03.

---

## Try this (REPL experiments)

Each probe builds a diagnostic reflex you'll lean on in ex03b and beyond.

**REPL, exercise repo root.** Connect (`clojure -M:nrepl` then your editor, or plain `clojure`):

```clojure
(require '[exercises.ex03-guards :as ex03] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
(require '[com.fulcrologic.statecharts.data-model.operations :as op])
```

### Probe 1: What does `op/assign` actually return?

**Scenario.** You see `(op/assign :password "secret")` in test code and wonder what kind of value it is — a closure? A side effect? Something else?

```clojure
(op/assign :password "secret")
```

Expected:

```
=> {:op :assign, :data {:password "secret"}}
```

Just a map. The op is a **descriptor** — data that the runtime interprets. Calling `op/assign` doesn't mutate anything; only feeding the descriptor into `goto-configuration!`, an `on-entry` action, or a `script` body produces the actual assignment.

**Why this probe exists.** If a learner thinks `op/assign` is a "do this now" function, they'll be confused when it doesn't change the data model on its own. Knowing it's a descriptor makes the whole data-model system click.

### Probe 2: What does `data` look like inside a guard?

**Scenario.** You want to confirm exactly what's in `data` when your guard runs — top-level keys, what `:_event` looks like, whether there's anything else.

```clojure
(def captured (atom nil))
(def chart-with-spy
  (com.fulcrologic.statecharts.chart/statechart {}
    (com.fulcrologic.statecharts.elements/state {:id :login}
      (com.fulcrologic.statecharts.elements/transition
        {:event  :submit
         :cond   (fn [env data]
                   (reset! captured data)
                   false)             ; force fallback
         :target :authenticated})
      (com.fulcrologic.statecharts.elements/transition
        {:event :submit :target :rejected}))
    (com.fulcrologic.statecharts.elements/state {:id :authenticated})
    (com.fulcrologic.statecharts.elements/state {:id :rejected})))

(def env (t/new-testing-env {:statechart chart-with-spy
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/goto-configuration! env [(op/assign :password "secret")] #{:login})
(t/run-events! env :submit)
@captured
```

Expected (approximately):

```
=> {:password "secret"
    :_event  {:type :external
              :name :submit
              :data {}
              :com.fulcrologic.statecharts/event-name :submit}}
```

The session data appears at top level (`:password "secret"`). The current event sits under `:_event`. The event's *own* data (the payload) is at `(:data :_event)`.

**Why this probe exists.** Knowing the shape of `data` makes you fluent with guards. When your guard returns the wrong thing, the first check is "what's actually in `data` right now?" — this probe is how you answer that without guessing.

### Probe 3: Event payload via `:_event`

**Scenario.** You realize the password could come *with the event* instead of being set in session data first.

```clojure
(def env2 (t/new-testing-env {:statechart chart-with-spy
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(t/run-events! env2 {:name :submit :data {:password "secret"}})
@captured
```

Expected:

```
=> {:_event {:type :external
             :name :submit
             :data {:password "secret"}
             ...}}
```

(No top-level `:password` because nothing put it in session data; but `:_event :data :password` is `"secret"`.)

A guard that reads from the event payload looks like:

```clojure
(fn [_env data] (= "secret" (get-in data [:_event :data :password])))
```

**Why this probe exists.** Real Fulcro apps usually send the password *with* the submit event, not seed session data first. Knowing both forms exist makes the production code path readable.

### Probe 4: Guard short-circuit — second guard not called

**Scenario.** You're debugging a chart with several guards on the same event and you want to confirm only the first match runs.

```clojure
(def first-calls (atom 0))
(def second-calls (atom 0))
(def short-circuit-chart
  (com.fulcrologic.statecharts.chart/statechart {}
    (com.fulcrologic.statecharts.elements/state {:id :a}
      (com.fulcrologic.statecharts.elements/transition
        {:event :go :cond (fn [_ _] (swap! first-calls inc) true) :target :b})
      (com.fulcrologic.statecharts.elements/transition
        {:event :go :cond (fn [_ _] (swap! second-calls inc) true) :target :c}))
    (com.fulcrologic.statecharts.elements/state {:id :b})
    (com.fulcrologic.statecharts.elements/state {:id :c})))

(def env3 (t/new-testing-env {:statechart short-circuit-chart
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
(t/run-events! env3 :go)
[@first-calls @second-calls]
;; => [1 0]
```

Expected: `[1 0]`. The first guard ran once; the second was never consulted. The chart ended in `:b`.

**Why this probe exists.** Short-circuit semantics are why "more specific guard first, fallback second" works. Knowing it's *truly* short-circuit (not "evaluate all, pick highest") means you can reason about guard side effects (don't have them) and about expensive guards (put the cheap one first).

---

## Try breaking it

Three deliberate edits to your finished chart.

### Break 1: Put the unguarded transition first

**Edit.** Reorder the two `:submit` transitions inside `:login` so the unguarded one comes first:

```clojure
(state {:id :login}
  (transition {:event :submit :target :rejected})                     ; ← first
  (transition {:event  :submit
               :cond   (fn [_env data] (= (:password data) "secret"))
               :target :authenticated}))
```

**Predicted symptom.** Assertion 2 (correct password → `:authenticated`) now fails. The unguarded transition matches first; the guarded one is never consulted; the chart always lands in `:rejected`.

**What this proves.** Document order is the tiebreaker. Specific-guarded-first / unguarded-fallback-second is a *positional* idiom, not an enforced behavior.

**Restore the original order** before continuing.

### Break 2: Remove the `:cond` from the first transition

**Edit.** Drop the guard, leaving two unconditional transitions:

```clojure
(state {:id :login}
  (transition {:event :submit :target :authenticated})       ; ← no more cond
  (transition {:event :submit :target :rejected}))
```

**Predicted symptom.** Assertions 1 and 2 still pass (`:authenticated` is the first match). Assertion 3 (wrong password → `:rejected`) fails — with no guard on the first transition, password-`"wrong"` *also* fires the first transition and lands in `:authenticated`.

**What this proves.** A missing `:cond` means *unconditionally enabled*. The runtime treats absent as "always true." This is what makes the fallback pattern terse: you don't write `:cond (fn [_ _] true)` on the fallback; you just omit `:cond`.

**Restore the guard** before continuing.

### Break 3: Make the guard return the *string* "true" instead of `true`

**Edit.** Wrap the guard's return value in `str`:

```clojure
(transition {:event  :submit
             :cond   (fn [_env data] (str (= (:password data) "secret")))   ; returns "true" / "false"
             :target :authenticated})
```

**Predicted symptom.** *Both* `"true"` and `"false"` are truthy in Clojure — only `false` and `nil` are falsy. So `"false"` (returned when the password is wrong) still counts as truthy, and the guarded transition fires regardless. Assertion 3 fails because wrong-password also reaches `:authenticated`.

**What this proves.** Guards are truthy/falsy, not boolean-strict. Anything that isn't `false` or `nil` makes the transition fire. Real bugs that hit this: returning a string "ok"/"no", returning a non-empty collection, accidentally not returning the comparison.

**Restore the guard** before continuing.

---

## Common breakages

Symptom-indexed.

### Assertion 1 fails: `(t/goto-configuration! …)` throws "Invalid configuration!"

`:login` doesn't exist in your chart yet — `goto-configuration!` validates that the leaf states it's asked to set are actually defined. Add `(state {:id :login})` (Step 4.1 above) before debugging anything else.

### Assertion 2 fails: chart ends in `:rejected` even with the right password

Two likely causes:

1. **The unguarded transition is declared first.** Open the file and confirm the guarded `:cond` transition is *above* the unguarded fallback. Document order matters; first match wins (per [tutorial Step 2](./tutorial.md)).
2. **Your `:cond` returns a wrong value.** Add `(println "cond called with" data)` inside the guard temporarily, run the test, and confirm `data` contains `:password "secret"`. If the println never fires, your transition's `:event` doesn't match — check spelling.

### Assertion 3 fails: chart ends in `:authenticated` even with the wrong password

The opposite problem from #2. Two likely causes:

1. **You forgot the `:cond` on the first transition.** Without a guard, the first transition fires unconditionally regardless of the password.
2. **Your `:cond` returns a non-boolean truthy value.** See Break 3 above. If your guard does `(str ...)` or returns a non-empty collection or just `data`, the result is truthy and the transition fires when you wanted it not to.

### Assertion 4 fails: `:retry` from `:rejected` doesn't return to `:login`

The `:retry` transition isn't declared, or it's declared on the wrong state. The transition must live *inside* `(state {:id :rejected} ...)`; declaring it on `:login` or at the chart root won't work (the latter throws — see ex02 runbook common breakages).

### Guard's `data` is `nil` and `(:password data)` returns nil

You called `(t/run-events! env :submit)` *without* having set up the data model. `goto-configuration!` is what seeds it; without that call (or an `on-entry` `op/assign`), `data` is `nil`. `(:password nil)` is `nil`, which is falsy, so the guard fails silently and the chart falls through to the fallback. Always set up session data before driving the chart in scenarios that depend on it.

### Guard throws an exception — and the chart still seems to "work"

Surprising but real: a guard that throws is treated by the runtime as "guard returned false." The exception is logged (visible in the DEBUG output) but doesn't crash the chart — the runtime falls through to the next matching transition. This is a footgun. See [`gotchas.md` #2](./gotchas.md).

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex03_guards.cljc` requires `statecharts.elements` and refers `state` and `transition`.
- [ ] `login-gate` has a `:login` state with **two** `:submit` transitions — the guarded one first, the unguarded one second.
- [ ] `:authenticated` and `:rejected` are top-level sibling states of `:login`.
- [ ] `:rejected` has a `:retry → :login` transition.
- [ ] `clojure -M:test --focus exercises.ex03-guards/guards-test` reports `4 assertions, 0 failures.`.
- [ ] You can recite the guard signature without looking: `(fn [env data] -> boolean)`.
- [ ] You can predict what happens if you swap the transitions' order (Break 1) and why.

**Explain in one breath:**

> *"On `:submit`, the runtime tries the guarded transition first; if its `:cond` returns truthy, it fires and the chart lands in `:authenticated`. Otherwise the runtime falls through in document order to the unguarded transition, which always fires, sending the chart to `:rejected`. `:retry` from `:rejected` returns to `:login`."*

If that's natural, ex03 is internalized. Move on to [`quiz.md`](./quiz.md), check [`gotchas.md`](./gotchas.md) for the silent footguns, then ex03b (data operations — going deeper on `op/assign` and on-entry).
