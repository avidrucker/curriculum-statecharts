# Runbook — ex01: Compound States & Configuration

The operational pass. By the end of this document you'll have edited `src/exercises/ex01_compound_states.cljc` to define a working traffic-light chart and seen all four assertions go green.

**Estimated time:** 15–25 minutes if you've completed [`setup.md`](../setup.md); 45 minutes if you've never used a Clojure REPL before.

---

## Prerequisites

- You've worked through [`setup.md`](../setup.md) and have `clojure -M:test --focus :solutions` reporting green.
- You've read [`tutorial.md`](./tutorial.md) for ex01. You don't need to memorize it — but the runbook references its terms without re-defining them.

The cross-cutting [`glossary.md`](../glossary.md) is the lookup table if any term in this runbook is unfamiliar.

---

## Where commands run

This runbook uses two terminals on the **exercise repo root** (`statechart-exercises`, the sibling of this curriculum repo). All paths below are relative to that root.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` — the interactive Clojure REPL where you load namespaces and run tests | `clojure -M:nrepl` then connect from your editor, *or* `clojure` for a plain prompt (no editor integration) |
| **Shell** | `clojure -M:test` — the kaocha test runner, one-shot or `--watch` | `clojure -M:test --focus exercises.ex01-compound-states/traffic-light-test` |

Most learners will work primarily in the REPL. The shell is a fallback if your editor isn't wired up.

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex01_compound_states.cljc` in your editor. The file you'll be editing is ~40 lines:

```clojure
(ns exercises.ex01-compound-states
  "Exercise 1: Compound States & Configuration
   Goal: Build a traffic light that cycles through red → green → yellow → red.
   The light starts in :red.
   Topics: compound states, transitions, events, configuration
   ...")
```

Three pieces to notice before touching anything:

1. **The `(ns …)` form.** It requires `clojure.test`, `statecharts.chart`, and `statecharts.testing`. *It does not yet require `statecharts.elements`* — that's the namespace exposing `state` and `transition`. You'll need to add it in Step 3.
2. **The `;; TODO:` comment**, just above `(def traffic-light …)`. This is the chart you'll build. The body is currently `;; YOUR CODE HERE` — an empty `statechart` invocation.
3. **The `deftest traffic-light-test`** below. Four `testing` blocks, four `is (t/in? …)` assertions. *This is the executable specification.* If you can satisfy these four, you've solved the exercise.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Open one terminal, start a watch-mode test runner so you can see results as you save:

```sh
clojure -M:test --focus exercises.ex01-compound-states/traffic-light-test --watch
```

The runner produces a lot of output — every failing assertion dumps the entire testing-env map (~2500 chars per failure), and DEBUG-level logging from the library is interleaved between assertions. **Don't try to read all of it.** Skip down to the summary line at the bottom:

```
1 tests, 4 assertions, 4 failures.
```

That's your baseline. The empty `statechart` doesn't have a `:red` state, so the `t/in?` check returns false for every assertion. (When tests fail, the format is `… N failures.` — no trailing `, 0 errors`. The `errors` count only appears in the line when actual JVM exceptions were thrown.)

Worth knowing: **an empty `(statechart {})` is a legal chart that builds and starts without complaint** — the runtime just lands in an empty configuration (`#{}`). That's what's happening before you write your first state: the chart builds, the env starts, no exception is raised — the four assertions just fail because the configuration is empty. See [gotchas #1](./gotchas.md) for the implications.

> **Tip for noisy output.** If the DEBUG logging is bothering you while iterating, you can quiet it by adjusting the timbre logging level in a `dev/logback-test.xml`, or by using `--no-capture-output --reporter kaocha.report/documentation` for a tidier presentation. Neither is required to solve the exercise — it's a paper-cut, not a blocker.

Keep this terminal open for the rest of the runbook. Every code change you make to the exercise file will trigger a re-run.

---

## Step 3 — Add the elements require

The exercise stub doesn't import `state` or `transition` — you have to add the require yourself.

Edit `src/exercises/ex01_compound_states.cljc`. The current `(ns …)` form ends like this:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Add a fourth require line for the elements namespace:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. The watch-mode test runner will re-run and **still report 4 failures** — that's expected, you haven't added the chart yet. What you've prevented is the `Unable to resolve symbol: state` error that would otherwise blow up the moment you started typing the chart body.

> **Why this isn't already in the stub.** The exercise file is a *minimal* starting point — it imports only what the (yet-to-exist) chart strictly requires. Adding `state` and `transition` to your namespace is part of the muscle memory the early exercises build.

---

## Step 4 — Build the chart, one state at a time

You'll do this in three small edits, with the test re-running between each. The point isn't to type fast — it's to see *which test passes when*, so you can map a chart change to a concrete assertion.

### 4.1 — Add the `:red` state

Replace `;; YOUR CODE HERE` with a single state:

```clojure
(def traffic-light
  (statechart {}
    (state {:id :red})))
```

Save. The watch-mode runner re-runs.

Expected — **the first assertion now passes, the other three still fail**. Skip past the per-assertion env dumps and confirm the summary line at the bottom:

```
1 tests, 4 assertions, 3 failures.
```

Why? Because the chart now has a `:red` state, document order makes it the initial state, and `(t/start! env)` lands the chart there. `(t/in? env :red)` returns truthy. The other three tests still fail because there's no `:green`, `:yellow`, or `:next` transition.

This is the smallest possible "the chart works" milestone. Hold onto the mental model: *one state, no transitions, and you've satisfied an assertion*.

### 4.2 — Add the `:red → :green` transition and the `:green` state

Extend the chart:

```clojure
(def traffic-light
  (statechart {}
    (state {:id :red}
      (transition {:event :next :target :green}))
    (state {:id :green})))
```

Save. Re-run.

Expected — **two assertions pass, two still fail**:

```
1 tests, 4 assertions, 2 failures.
```

The `:next` event fires while the chart is in `:red`, the transition matches, and the chart moves to `:green`. But there's no `:next` transition out of `:green` yet, so the third assertion still fails.

### 4.3 — Complete the cycle

Add the final two transitions and the `:yellow` state:

```clojure
(def traffic-light
  (statechart {}
    (state {:id :red}
      (transition {:event :next :target :green}))
    (state {:id :green}
      (transition {:event :next :target :yellow}))
    (state {:id :yellow}
      (transition {:event :next :target :red}))))
```

Save. Re-run.

Expected — **all four assertions pass**. The output is suddenly much smaller, because there are no failures to dump:

```
1 tests, 4 assertions, 0 failures.
```

The cycle is closed: `:red` → `:green` → `:yellow` → `:red`. The last transition (`:yellow` → `:red`) is what makes the test name "cycles" meaningful.

You've solved ex01.

---

## Try this (REPL experiments)

The chart works. Now build the diagnostic reflexes that will matter on every later exercise. Each experiment below is a probe — a question you'd ask the REPL when something later goes sideways.

**REPL, exercise repo root.** Connect to a REPL (`clojure -M:nrepl` then your editor, or `clojure` for plain), then load the namespace:

```clojure
(require '[exercises.ex01-compound-states :as ex01] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: What does the chart definition look like as data?

**Scenario.** You're going to write more complex charts and you want to know what the `statechart` factory actually returns.

```clojure
(:id ex01/traffic-light)        ; → :ROOT
(:node-type ex01/traffic-light) ; → :statechart
(:children ex01/traffic-light)  ; → a vector of child IDs
```

Expected: `:id` is `:ROOT` (auto-assigned by the factory), `:node-type` is `:statechart`, and `:children` is a vector of keyword IDs. **Note:** the vector will likely contain *more* than the three state IDs you declared — the factory auto-inserts a generated initial element (something like `:initial25839`) pointing at the first state. That's library bookkeeping; you don't interact with it directly.

For the full data dump:

```clojure
(clojure.pprint/pprint ex01/traffic-light)
```

It's long and deeply nested. You don't need to understand every key — what matters is that *the chart is data*, inspectable like any Clojure value.

**Why this probe exists.** Later exercises (especially ex03c, convenience macros) use the same kind of data. If you can `pprint` a chart, you can diff two charts, you can introspect them with `tree-seq`, you can generate them. The chart's data-ness is load-bearing.

### Probe 2: What does the configuration look like before vs. after `start!`?

**Scenario.** You're worried `start!` didn't do what you think it did.

```clojure
(def env (t/new-testing-env
           {:statechart ex01/traffic-light
            :mocking-options {:run-unmocked? true}}
           {}))
(t/in? env :red)        ; → false   (configuration is nil — chart hasn't started)
(t/start! env)
(t/in? env :red)        ; → true    (configuration is now #{:red})
(t/in? env :ROOT)       ; → false   (the auto-generated root is NOT in the configuration)
```

Expected: the first `t/in?` returns `false` (the chart's working memory doesn't have a configuration yet; `contains?` on `nil` returns `false`). After `start!`, `:red` is in the configuration set, but `:ROOT` is not — the root node is the chart's container, not a state the chart can be "in."

**Why this probe exists.** If you ever see a test that fails with `(t/in? env :something)` on the first assertion, the first question to ask is "did `start!` run?" This experiment is the calibration that makes that question instinctive. The `:ROOT` check is the second calibration: knowing that the root is never in the configuration saves you a confusing debugging session in later exercises.

### Probe 3: Firing an unknown event

**Scenario.** You typo an event name (`:nextt` instead of `:next`) and the test fails silently — no error, just a wrong configuration.

```clojure
(def env (t/new-testing-env
           {:statechart ex01/traffic-light
            :mocking-options {:run-unmocked? true}}
           {}))
(t/start! env)
(t/run-events! env :bogus)   ; nothing matches; chart should stay in :red
(t/in? env :red)             ; → true
(t/in? env :green)           ; → false
```

Expected: the chart stays in `:red`. No error is thrown. The event is consumed and dropped.

**Why this probe exists.** Statecharts don't throw on unknown events — they ignore them. This is a feature (charts are robust to spurious events) but also a hazard (typos look like correct events at first glance). If your test fails because the chart didn't move, your first question is "did the event actually match a transition?" — and this probe is how you check.

---

## Try breaking it

Deliberate edits to your finished chart to see what goes wrong. Each one builds intuition about a rule that will matter later.

### Break 1: Swap document order

**Edit.** In `src/exercises/ex01_compound_states.cljc`, move the `:green` state above `:red`:

```clojure
(def traffic-light
  (statechart {}
    (state {:id :green}                      ; ← now first
      (transition {:event :next :target :yellow}))
    (state {:id :red}
      (transition {:event :next :target :green}))
    (state {:id :yellow}
      (transition {:event :next :target :red}))))
```

**Predicted symptom.** The first assertion (`starts in :red`) now fails — the chart starts in `:green` because it's now the first child in document order.

**What this proves.** Without an explicit `:initial`, the chart root picks the first child in document order. This is the rule the tutorial named in Step 4.

**Restore the original ordering** (`:red`, `:green`, `:yellow`) before continuing.

### Break 2: Drop the cycle

**Edit.** Remove the transition inside `:yellow`:

```clojure
(state {:id :yellow})    ; ← no transition
```

**Predicted symptom.** The fourth assertion (`:next goes yellow → red (cycles)`) fails. The chart reaches `:yellow` correctly, but the next `:next` event has no matching transition out of `:yellow`, so the chart stays in `:yellow`.

**What this proves.** A "missing transition" doesn't error — the chart just doesn't move. Probe 3 above is the diagnostic instinct that catches this.

**Restore the `:yellow` → `:red` transition** before continuing.

---

## Common breakages

Symptom-indexed. If your test failures don't match the expectations above, find your symptom here.

### `Unable to resolve symbol: state`

You skipped Step 3. The `state` and `transition` symbols come from `com.fulcrologic.statecharts.elements`, which is *not* imported by the exercise stub by default. Add it to the `(:require …)` block in the `(ns …)` form.

### `clojure.lang.ExceptionInfo: Invalid chart` (or similar)

Often a typo — `:targe` instead of `:target`, `:event/` instead of `:event`. Check the keywords in the transition. Compare against the solution at `src/solutions/ex01_compound_states.cljc` for the exact keyword shape.

### All four assertions fail and you've definitely added the chart

Three things to check, in order:

1. **Did the file save?** Some editors silently fail to write `.cljc` files. Reopen the file in a plain text editor to confirm.
2. **Did the test runner pick up the change?** If you're not in `--watch` mode, you need to re-invoke `clojure -M:test …`. If you are in `--watch` mode, look for the "Reloading namespace…" line — if it's missing, the watcher isn't seeing your save.
3. **Did the namespace reload in the REPL?** REPLs cache namespaces. If you defined `traffic-light` in the REPL, you need `(require '[exercises.ex01-compound-states] :reload)` to pick up file changes. Without `:reload`, the REPL uses the version it loaded the first time.

### Three assertions pass, one fails (`:next goes yellow → red (cycles)`)

You have the first three transitions but the last one targets the wrong state or is missing. Check `:yellow`'s transition: `:target` should be `:red`, not `:red-state` or `:start`.

### `t/in?` returns `nil` for every check

You forgot `(t/start! env)`. The chart isn't running. See Probe 2 above for the calibration session.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex01_compound_states.cljc` requires `com.fulcrologic.statecharts.elements` and refers `state` and `transition`.
- [ ] `traffic-light` is defined as a `(statechart {} …)` with three states.
- [ ] Each state has an `:id` (`:red`, `:green`, `:yellow`) and one `transition` with `:event :next` and the correct `:target`.
- [ ] `clojure -M:test --focus exercises.ex01-compound-states/traffic-light-test` reports `4 assertions, 0 failures.`.
- [ ] You can run Probes 1, 2, and 3 in the REPL without consulting the runbook.
- [ ] You can predict what would happen if you reordered the states or dropped a transition, without re-running.

**Explain in one breath:**

> *"The chart's three top-level states are children of an implicit compound root. Document order picks `:red` as the initial state. Each state declares one `:next` transition targeting the next color, and `:yellow`'s transition targets `:red` to close the cycle. `t/in?` queries the current configuration after each event."*

If that's natural, ex01 is internalized. Move on to [`quiz.md`](./quiz.md) and then ex02.
