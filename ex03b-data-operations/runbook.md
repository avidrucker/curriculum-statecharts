# Runbook — ex03b: Data Model Operations & Event Data

The operational pass. By the end you'll have edited `src/exercises/ex03b_data_operations.cljc` to define a counter chart and seen all seven assertions go green.

**Estimated time:** 30–45 minutes if you've completed ex03. The chart is *smaller* than ex03's (one state vs. three), but introduces five new mechanics: `on-entry`, `script`, the vector-of-ops contract, `handle`, and event-payload reading.

---

## Prerequisites

- You've completed ex03 — guards, `op/assign`, `goto-configuration!`, and the structure of `data` (session keys + `:_event`) are all familiar.
- You've read [`tutorial.md`](./tutorial.md) for ex03b. The runbook references its terms (`on-entry`, `script`, targetless transition, vector-of-ops, `handle`) without re-defining them.

---

## Where commands run

Same setup as previous exercises:

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` — interactive REPL | from the exercise repo root |
| **Shell** | `clojure -M:test` — the kaocha test runner | from the exercise repo root |

All paths below are relative to `statechart-exercises/`.

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex03b_data_operations.cljc`. ~82 lines. Five things to notice:

1. **The docstring spells out the patterns.** It quotes the exact shape of `on-entry`, `handle`, and event-data access. Read it slowly — most of what you need is right there.
2. **The variable is named `cart-chart`** but the topic is a counter. Cosmetic carryover from an earlier exercise version. Don't rename it — the test references this name.
3. **The `(ns …)` form** requires `clojure.test`, `statecharts.chart`, and `statecharts.testing` — but **not** `statecharts.elements`, **not** `statecharts.convenience`, **not** `statecharts.data-model.operations`. You'll add all three.
4. **The `deftest`** has seven assertions in one `let` block (vs ex03's two blocks). The chart is reusable across all scenarios because no scenario terminates the chart; every event leaves the chart in `:active`.
5. **The tests use `(t/data env)`** — new in this exercise — to inspect the data model directly. Returns the current data-model map.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex03b-data-operations/data-operations-test --watch
```

Skip the noisy per-assertion env dumps. Summary line:

```
1 tests, 8 assertions, 8 failures.
```

(Eight assertions, not seven — the first `testing` block has two `is` calls.)

Keep the watch terminal open.

---

## Step 3 — Add three requires

The exercise stub needs three additions to its `(:require …)` form:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.convenience :refer [handle]]
    [com.fulcrologic.statecharts.data-model.operations :as ops]
    [com.fulcrologic.statecharts.elements :refer [on-entry script state]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Three things to highlight:

- The alias is `ops` (plural), not `op` (singular). ex03 used `op`; ex03b uses `ops`. Both work — the namespace is the same `com.fulcrologic.statecharts.data-model.operations`; the alias is your choice. The exercise's docstring and solution both use `ops`, so we match.
- `state` is here but **not** `transition` — because the solution uses `handle` (from convenience) for every transition, never the raw `transition` element. You *could* refer `transition` too; you just don't need to.
- `on-entry`, `script` are new elements coming from the elements namespace.

Save. The runner re-runs and still reports `8 failures`.

---

## Step 4 — Build the chart, layer by layer

Six edits this time. The first one is the largest (adds the state + initialization); subsequent edits add one handler each.

### 4.1 — The `:active` state with `:count` initialization

Replace `;; YOUR CODE HERE`:

```clojure
(def cart-chart
  (statechart {}
    (state {:id :active}
      (on-entry {}
        (script {:expr (fn [_ _] [(ops/assign :count 0)])})))))
```

Save.

Expected — **first two assertions pass** (`starts in :active` and `count is 0`):

```
1 tests, 8 assertions, 6 failures.
```

What just happened:

1. `(t/start! env)` ran the chart. The chart entered `:active`.
2. `on-entry` fired. Its `script`'s `:expr` ran with `data = nil`.
3. The expr returned `[(ops/assign :count 0)]` — a vector with one operation descriptor.
4. The runtime interpreted the op and wrote `:count 0` into the session data.
5. `(:count (t/data env))` reads `0`.

**The vector wrapping is mandatory.** If you write `(fn [_ _] (ops/assign :count 0))` without the brackets, the test will *still fail* at the second assertion — silently, with `(:count (t/data env))` returning `nil`. See [Try breaking it #1](#try-breaking-it).

### 4.2 — Add `:increment`

```clojure
(state {:id :active}
  (on-entry {}
    (script {:expr (fn [_ _] [(ops/assign :count 0)])}))

  (handle :increment
    (fn [_ data] [(ops/assign :count (inc (:count data)))])))
```

Save.

Expected — **two more assertions pass** (`:increment increases count` and `multiple increments accumulate`):

```
1 tests, 8 assertions, 4 failures.
```

The `handle` form expands to a targetless transition with a script body. When `:increment` arrives:

1. The runtime finds the matching transition on `:active`.
2. The transition has no `:target`, so the chart stays in `:active`.
3. The script's `:expr` runs with `data = {:count 0, :_event {...}}`.
4. It returns `[(ops/assign :count (inc 0))]`.
5. The runtime writes `:count 1`.

The third assertion (`multiple increments accumulate`) passes because `:active` stays active across calls — each `:increment` increments from the current count.

### 4.3 — Add `:decrement` (with floor at 0)

```clojure
(handle :decrement
  (fn [_ data] [(ops/assign :count (max 0 (dec (:count data))))]))
```

(Inserted as a sibling of the `:increment` handler.)

Save.

Expected — **two more assertions pass** (`:decrement decreases count` and `:decrement floors at 0`):

```
1 tests, 8 assertions, 2 failures.
```

The `(max 0 (dec n))` form is the floor: if `:count` is `0`, `(dec 0)` is `-1`, and `(max 0 -1)` is `0`. If `:count` is `3`, `(dec 3)` is `2`, and `(max 0 2)` is `2`. Common one-liner; worth keeping in your kit.

### 4.4 — Add `:set-value` (read from event payload)

```clojure
(handle :set-value
  (fn [_ data]
    (let [v (-> data :_event :data :value)]
      [(ops/assign :count (or v 0))])))
```

Save.

Expected — **one more assertion passes**:

```
1 tests, 8 assertions, 1 failures.
```

The path `(-> data :_event :data :value)`:

- `data` — the function arg.
- `:_event` — the runtime-injected current event.
- `:data` — the payload key on the event map.
- `:value` — the application key.

The test fires `{:name :set-value :data {:value 42}}`, so `(-> data :_event :data :value)` is `42`. The `(or v 0)` is paranoid — if a `:set-value` event arrived without a `:value` field, we'd default to 0 rather than assigning `nil` to `:count` (which would make `(inc :count)` blow up later).

### 4.5 — Add `:reset`

```clojure
(handle :reset
  (fn [_ _] [(ops/assign :count 0)]))
```

Save.

Expected — **all eight assertions pass**:

```
1 tests, 8 assertions, 0 failures.
```

Note that `:reset`'s expr does exactly what `on-entry`'s does. That's intentional — "reset to initial state" is "rerun the initialization." Some charts factor this out into a helper; for ex03b, two copies of `(ops/assign :count 0)` is fine.

You've solved ex03b.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect and load:

```clojure
(require '[exercises.ex03b-data-operations :as ex03b] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: What's `(t/data env)` at each point?

**Scenario.** You're debugging a chart and want a quick way to see what the data model contains right now.

```clojure
(def env (t/new-testing-env {:statechart ex03b/cart-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/data env)                              ; → nil   (chart not started)
(t/start! env)
(t/data env)                              ; → {:count 0}
(t/run-events! env :increment)
(t/data env)                              ; → {:count 1}
(t/run-events! env :increment :increment)
(t/data env)                              ; → {:count 3}
(t/run-events! env {:name :set-value :data {:value 99}})
(t/data env)                              ; → {:count 99}
```

Expected: `t/data` mirrors the chart's data model. The transition between `nil` (chart hadn't run) and `{:count 0}` (after `start!`) is the moment `on-entry`'s script applied its ops.

**Why this probe exists.** Every later exercise will lean on `t/data` for assertions about the data model. Knowing it returns `nil` before `start!` saves the "why is everything nil" debugging session.

### Probe 2: `ops/assign` returning a vector vs returning a single op

**Scenario.** You're about to write a `:expr` for a new chart and want to *see* what happens when you forget the brackets.

```clojure
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [state on-entry script]])
(require '[com.fulcrologic.statecharts.data-model.operations :as ops])

(def broken-chart
  (statechart {}
    (state {:id :active}
      (on-entry {}
        (script {:expr (fn [_ _] (ops/assign :count 0))})))))    ; NO BRACKETS

(def env (t/new-testing-env {:statechart broken-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/data env)                              ; → nil    (!)
```

Expected: `nil`. The chart starts, the on-entry script runs, the expr returns a *map* (not a vector), the runtime tries to iterate over it, nothing matches the op interpreter, the data model stays untouched. **No error, no warning.**

**Why this probe exists.** Once you've seen this with your own eyes, you'll never forget the brackets. The footgun deserves muscle memory.

### Probe 3: Inspect what `handle` returns

**Scenario.** You're curious whether `handle` is "magic" or just sugar.

```clojure
(require '[com.fulcrologic.statecharts.convenience :refer [handle]])
(handle :go (fn [_ _] []))
```

Expected (approximately):

```clojure
{:id :transitionXXXXX
 :event :go
 :target nil
 :type :external
 :node-type :transition
 :children [{:id :scriptXXXXX
             :expr #object[...]
             :node-type :script}]}
```

A `:transition` element with `:target nil` (targetless!) containing a `:script` child. Not magic — just sugar.

**Why this probe exists.** The convenience namespace is marked ALPHA. When you read someone else's chart that uses `handle`, you should be able to mentally rewrite it as `(transition + script)`. This probe makes the rewriting reflexive.

### Probe 4: Event-payload reading without `:_event`

**Scenario.** You wonder whether there's a shortcut for reading event data — maybe `env` has the event in it?

```clojure
(def env-keys-log (atom nil))
(def spy-chart
  (statechart {}
    (state {:id :active}
      (handle :inspect
        (fn [env data]
          (reset! env-keys-log {:env-keys (vec (keys env))
                                :data-keys (vec (keys data))})
          [])))))

(def env (t/new-testing-env {:statechart spy-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env {:name :inspect :data {:foo 42}})
@env-keys-log
```

Expected:

```clojure
{:env-keys [:com.fulcrologic.statecharts/working-memory-store
            :com.fulcrologic.statecharts/vwmem
            :com.fulcrologic.statecharts/event-queue
            ... (no event key) ...]
 :data-keys [:_event]}     ; the event is here, not in env
```

The event is **not** in `env`. It's in `data._event`. `env` is plumbing; `data` is content. (Same rule ex03's gotcha #3 covered.)

**Why this probe exists.** When debugging a chart and "where's the event?" comes up, the answer is *always* `data._event`. Never `env`. Make this reflexive.

---

## Try breaking it

### Break 1: Drop the vector wrapping

**Edit.** In any `:expr`, remove the outer brackets:

```clojure
(on-entry {}
  (script {:expr (fn [_ _] (ops/assign :count 0))}))    ; ← no brackets
```

**Predicted symptom.** The chart starts, the on-entry script runs without error, but `(t/data env)` returns `nil` and the second assertion fails (`(= 0 (:count (t/data env)))` becomes `(= 0 nil)` = false). All downstream assertions fail too because `:count` is `nil` and `(inc nil)` throws.

**What this proves.** Scripts must return a *vector* of ops. A single op (a map) is silently ignored. This is the canonical ex03b footgun.

**Restore the brackets** before continuing.

### Break 2: Read on-entry's `data` before it's been initialized

**Edit.** Change `on-entry`'s expr to read from `data`:

```clojure
(on-entry {}
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
```

**Predicted symptom.** First assertion fails with an exception or a `:count` of `nil`. `data` is `nil` on first entry (the script hasn't applied its own op yet); `(:count nil)` is `nil`; `(inc nil)` throws an NPE.

Wait — does the exception kill the test or get silently swallowed (like with guards)? Try it. *(Empirical prediction: it propagates and the test errors out. Guards have their own catch-and-treat-as-false handling; scripts probably don't.)*

**What this proves.** On-entry scripts run *before* their own ops are applied. The data model starts empty; reading from it on first entry will see `nil`. The defensive form is to *write* values, not read-and-modify, in initialization scripts. Or use `(or (:count data) 0)` to guard the read.

**Restore the pure-write form** before continuing.

### Break 3: Give `:increment` a `:target`

**Edit.** Replace `handle` with a `transition` that has a `:target`:

```clojure
(transition {:event :increment :target :active}             ; ← targets self!
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
```

**Predicted symptom.** The tests may *still pass*, but you'll notice in any logs that `on-entry` re-runs on every `:increment` — because targeting `:active` causes the chart to *exit and re-enter* `:active`. Each re-entry runs the on-entry script, resetting `:count` to 0 before the increment's own op runs. (Depending on op ordering and SCXML semantics, you might end up with `:count = 1` after every `:increment`, breaking the "accumulate" assertion.)

**What this proves.** Targetless transitions are *not* the same as "transition with target = same state." A self-target causes a real exit/entry cycle; a missing target leaves the chart in place. The counter wants the latter.

**Restore the `handle` form** before continuing.

---

## Common breakages

### Assertion 1 fails: `(= 0 (:count (t/data env)))` is false

Two likely causes:

1. **No on-entry, or on-entry's script doesn't apply an op.** Check that your `:active` state has `(on-entry {} (script {:expr ...}))` and that the script returns `[(ops/assign :count 0)]` (vector with the op).
2. **Vector wrapping forgotten.** The most common bug. If `(t/data env)` returns `nil`, the on-entry ran but the op didn't apply. Re-check that the expr's return is wrapped in `[...]`.

### Assertion fails: `(t/in? env :active)` is false

You forgot to declare the `:active` state, or you wrote `(state :active …)` instead of `(state {:id :active} …)`. The first arg to `state` is an attributes map.

### `:increment` doesn't change `:count`

Three possibilities:

1. **No `handle :increment` declared.** Check the chart definition.
2. **`handle` returned but the expr returns a non-vector.** Vector. Always vector. `[(ops/assign :count (inc (:count data)))]` — brackets.
3. **You used `(state ...)` instead of `(handle ...)`** by mistake. `handle` is a function call, not an element wrapper; it's used *inside* a state's body.

### `:set-value` doesn't read the value

Path issue. Trace through:

```clojure
(let [event-data (-> data :_event :data)
      value     (:value event-data)]
  (println "event-data:" event-data ", value:" value))
```

If `event-data` is `nil`, the event didn't carry data — check that the test calls `(t/run-events! env {:name :set-value :data {:value 42}})` (map form), not `(t/run-events! env :set-value)` (keyword-only form, no payload).

### NullPointerException out of the runtime

You read from `data` before it had what you expected. Most likely: on-entry trying to do `(inc (:count data))` on initial entry, or a handler running before any on-entry initialized the data model.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex03b_data_operations.cljc` requires `statecharts.elements` (state, on-entry, script), `statecharts.convenience` (handle), `statecharts.data-model.operations` (ops).
- [ ] `cart-chart` is defined with one `:active` state containing on-entry initialization and four `handle` event handlers.
- [ ] Every `:expr` returns a *vector* of ops, even when there's only one op.
- [ ] `clojure -M:test --focus exercises.ex03b-data-operations/data-operations-test` reports `8 assertions, 0 failures.`.
- [ ] You can predict the result of `(t/data env)` after each event without running the chart.
- [ ] You can rewrite a `handle` form as `(transition {:event ...} (script {:expr ...}))` and back.

**Explain in one breath:**

> *"`on-entry` runs a script that returns `[(ops/assign :count 0)]` to initialize the data model. Each `handle` is a targetless transition with a script body — when the event arrives, the script runs, returns a vector of ops, the runtime applies them to the data model, and the chart stays in `:active`. Event-time data is read via `(-> data :_event :data :key)`."*

If that's natural, ex03b is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex03c (convenience macros — going deeper on `handle` and friends).
