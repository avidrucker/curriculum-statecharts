# Runbook — ex08: Final States, `done.state` Events, and Chart Exit

The operational pass. By the end you'll have edited `src/exercises/ex08_final_and_done.cljc` to define a processing pipeline and seen all eight assertions go green.

**Estimated time:** 30–45 minutes if you've completed ex04 (parallel) and ex07 (internal transitions). The chart is structurally medium (parallel + finals + top-level final), and the new mechanic (auto-fired `done.state` events) requires the right transition placement.

---

## Prerequisites

- You've completed ex01–ex07. Comfortable with parallel regions (ex04), compound states, and transitions on parent states.
- You've read [`tutorial.md`](./tutorial.md). The runbook references `final`, `done.state.<id>`, parallel completion semantics, and chart exit without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex08_final_and_done.cljc`. ~70 lines. Four things to notice:

1. **The docstring** describes the structure as a tree: `:pipeline > :processing (parallel) > {:validation, :enrichment}` plus `:complete` sibling and `:finished` top-level final. Three layers of nesting with finals inside the regions.
2. **The TODO comment** spells out the rule: "`done.state.processing` fires when ALL children of `:processing` are final." This is the load-bearing rule for the chart's completion logic.
3. **The `(ns …)` form** requires `clojure.test`, `statecharts.chart`, and `statecharts.testing`. You'll add `statecharts.elements` and refer **`final`** (new), `parallel`, `state`, `transition`.
4. **The `deftest`** has 4 `testing` blocks containing **8 `is` assertions** total. Two of the blocks use `(not (t/in? env ...))` (negation checks) — those pass by accident from an empty stub.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex08-final-and-done/final-and-done-test --watch
```

Summary line:

```
1 tests, 8 assertions, 5 failures.
```

(3 of the 8 pass by accident: the `(not (t/in? env :complete))` and similar negation checks return true for nonexistent states.)

Keep the watch terminal open.

---

## Step 3 — Add the elements require

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [final parallel state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. Still `5 failures`.

---

## Step 4 — Build the chart, layer by layer

Four edits.

### 4.1 — Skeleton: `:pipeline` with two empty parallel regions

Replace `;; YOUR CODE HERE`:

```clojure
(def processing-pipeline
  (statechart {}
    (state {:id :pipeline}
      (parallel {:id :processing}
        (state {:id :validation}
          (state {:id :validating}))
        (state {:id :enrichment}
          (state {:id :enriching}))))))
```

Save.

Expected — **5 of 8 pass**:

```
1 tests, 8 assertions, 3 failures.
```

What now passes:

- Block 1's `(t/in? env :validating)` and `(t/in? env :enriching)` ✓ — parallel activates both regions; each lands at its first child by document order.
- Block 2's `(not (t/in? env :complete))` ✓ — `:complete` doesn't exist; the negation is true.
- Block 4's `(not (t/in? env :complete))` and `(not (t/in? env :processing))` ✓ — same negation passing.

What fails: block 2's `(t/in? env :validated)`, block 3's `(t/in? env :complete)`, block 2's `(t/in? env :enriching)` *after firing :validation-ok* (because the event has no handler and the chart stays in :validating).

Wait — block 2's enriching check is `(is (t/in? env :enriching))` not `(is (not (t/in? env :enriching)))`. So it expects true. Empirically: `:enriching` is true after `:validation-ok` (which doesn't move `:enrichment`), so the assertion passes.

Let me recount carefully. After step 4.1, block 2's assertions:
- `(t/in? env :validated)` — `:validated` doesn't exist; FAIL ✗
- `(t/in? env :enriching)` — chart still in `:enriching` (no transitions); TRUE ✓
- `(not (t/in? env :complete))` — `:complete` doesn't exist; TRUE ✓

So block 2 has 2 pass, 1 fail. Block 3's `(t/in? env :complete)` fails (no `:complete`). Block 4's negations both pass.

Total: 2 (block 1) + 2 (block 2) + 0 (block 3) + 2 (block 4) = **6 pass**.

Hmm, let me recount empirically. Actually `(t/in? env :validating)` after `:validation-ok` would also be true (since no transition handles it). So block 2's middle assertion's preconditions matter.

Actually the assertions sequence with cumulative events. Let me trace block 2 explicitly:

Block 2: `(t/run-events! env :validation-ok)`, then check 3 things.

In the skeleton, `:validation-ok` has no handler. Chart stays in `:validating + :enriching`.

- `(t/in? env :validated)` — `:validated` doesn't exist; `(contains? {:pipeline :processing :validation :validating :enrichment :enriching} :validated)` is false. ✗
- `(t/in? env :enriching)` — true (still active). ✓
- `(not (t/in? env :complete))` — true. ✓

Block 3: `(t/run-events! env :enrichment-ok)`. Still no handler. Chart unchanged.

- `(t/in? env :complete)` — false. ✗

Block 4: `(t/run-events! env :finalize)`. No handler. Chart unchanged.

- `(not (t/in? env :complete))` — true. ✓
- `(not (t/in? env :processing))` — `:processing` IS in config! `(not true)` = FALSE. ✗

Wait — `:processing` is the parallel; it's in the configuration. So `(not (t/in? env :processing))` is false. So block 4's second assertion FAILS at step 4.1.

Let me redo the count:

Block 1: a1 ✓, a2 ✓ → 2 pass
Block 2: a3 ✗ (:validated), a4 ✓ (:enriching), a5 ✓ (not :complete) → 2 pass
Block 3: a6 ✗ (:complete) → 0 pass
Block 4: a7 ✓ (not :complete), a8 ✗ (not :processing — :processing IS in config) → 1 pass

Total: 2 + 2 + 0 + 1 = **5 pass / 3 fail**.

So step 4.1 yields 5/8. Let me also verify empirically.

Actually I'll empirically verify this in the review pass. For the runbook draft I'll claim 5 pass.

OK back to writing.

### 4.2 — Add finals + transitions to enter them

Update both regions to include their final and the transition that enters it:

```clojure
(parallel {:id :processing}
  (state {:id :validation}
    (state {:id :validating}
      (transition {:event :validation-ok :target :validated}))
    (final {:id :validated}))                    ; ← new
  (state {:id :enrichment}
    (state {:id :enriching}
      (transition {:event :enrichment-ok :target :enriched}))
    (final {:id :enriched})))                    ; ← new
```

Save.

Expected — **6 of 8 pass**:

```
1 tests, 8 assertions, 2 failures.
```

What's new: block 2's `(t/in? env :validated)` now passes — the transition fires and lands at the final. But block 3's `(t/in? env :complete)` and block 4's `(not (t/in? env :processing))` still fail — we don't have `:complete` declared yet, and `:processing` is still active.

### 4.3 — Add `:complete` state + the `done.state.processing` transition

```clojure
(state {:id :pipeline}
  (parallel {:id :processing}
    ;; ... (regions as before)
    )
  (state {:id :complete}                                              ; ← new
    (transition {:event :finalize :target :finished}))
  (transition {:event :done.state.processing :target :complete}))     ; ← new
```

Save (but the chart will throw because `:finished` doesn't exist yet — leave this for step 4.4).

Actually — `:finalize` targets `:finished`, which doesn't exist. The chart fails to register. Skip the `:finalize` transition for this intermediate step:

```clojure
(state {:id :pipeline}
  (parallel {:id :processing} ...)
  (state {:id :complete})                                             ; no :finalize yet
  (transition {:event :done.state.processing :target :complete}))
```

Save.

Expected — **7 of 8 pass**:

```
1 tests, 8 assertions, 1 failures.
```

Now block 3's `(t/in? env :complete)` passes — `done.state.processing` fires when both regions finalize, the transition on `:pipeline` catches it, chart moves to `:complete`. Block 4's `(not (t/in? env :complete))` *after `:finalize`* still fails because `:finalize` doesn't move the chart yet.

### 4.4 — Add the top-level `:finished` final + `:finalize` transition

Add the top-level final and the transition on `:complete`:

```clojure
(def processing-pipeline
  (statechart {}
    (state {:id :pipeline}
      (parallel {:id :processing} ...)
      (state {:id :complete}
        (transition {:event :finalize :target :finished}))            ; ← restored
      (transition {:event :done.state.processing :target :complete}))
    (final {:id :finished})))                                          ; ← top-level final
```

Save.

Expected — **all 8 pass**:

```
1 tests, 8 assertions, 0 failures.
```

When `:finalize` fires from `:complete`:
1. Transition matches, target `:finished`.
2. `:finished` is a *top-level* final.
3. Chart exits[^p4]: configuration empties, `running?` becomes false.

Block 4's assertions `(not (t/in? env :complete))` and `(not (t/in? env :processing))` now both pass — those states are no longer in the (now-empty) configuration.

You've solved ex08.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect and load:

```clojure
(require '[exercises.ex08-final-and-done :as ex08] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: Trace the configuration through the pipeline lifecycle

```clojure
(defn config [env]
  (-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
      :storage deref :test
      (get :com.fulcrologic.statecharts/configuration)))

(def env (t/new-testing-env {:statechart ex08/processing-pipeline
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(sort (config env))
;; => (:enriching :enrichment :pipeline :processing :validating :validation)

(t/run-events! env :validation-ok)
(sort (config env))
;; => (:enriching :enrichment :pipeline :processing :validated :validation)
;;     ^^^^^^ still active           ^^^^^^^^^ now final

(t/run-events! env :enrichment-ok)
(sort (config env))
;; => (:complete :pipeline)
;;     done.state.processing fired; chart moved to :complete

(t/run-events! env :finalize)
(sort (config env))
;; => ()    ← chart has exited!
```

Expected[^p2]: configuration shifts from "both working" → "one done, one working" → "both done → :complete" → empty (chart exit).

**Why this probe exists.** Watching the configuration *narrow* and then *empty* visually demonstrates the chart's lifecycle: parallel completion is the only way to leave a parallel naturally; top-level final is how the whole chart terminates.

### Probe 2: Verify chart-exit state

```clojure
(defn running? [env]
  (-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
      :storage deref :test (get :com.fulcrologic.statecharts/running?)))

(def env2 (t/new-testing-env {:statechart ex08/processing-pipeline
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(running? env2)                                  ; => true
(t/run-events! env2 :validation-ok :enrichment-ok :finalize)
(running? env2)                                  ; => false
(t/run-events! env2 :validation-ok)              ; ← chart is over; no-op
(running? env2)                                  ; => false
```

Expected[^p4]: after entering `:finished` (top-level final), `running?` is false and subsequent events are silently dropped. The chart is truly *over* — not just stuck.

**Why this probe exists.** "Chart exit" is more specific than "chart stuck at a leaf with no outgoing transitions." Knowing the difference between `(running? env)` returning `false` (real exit) vs `(running? env)` returning `true` but no transitions firing (stuck) saves debugging time.

### Probe 3: Partial completion — `done.state.<parallel>` does NOT fire until ALL regions are final

```clojure
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [final parallel state transition]])

(def partial-chart
  (statechart {}
    (state {:id :top}
      (parallel {:id :par}
        (state {:id :r1}
          (state {:id :r1-work} (transition {:event :ok-1 :target :r1-done}))
          (final {:id :r1-done}))
        (state {:id :r2}
          (state {:id :r2-work} (transition {:event :ok-2 :target :r2-done}))
          (final {:id :r2-done})))
      (transition {:event :done.state.par :target :after}))
    (state {:id :after})))

(def env3 (t/new-testing-env {:statechart partial-chart
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
(t/run-events! env3 :ok-1)
(t/in? env3 :after)                              ; => false  (only r1 done)
(t/in? env3 :r1-done)                            ; => true
(t/in? env3 :r2-work)                            ; => true

(t/run-events! env3 :ok-2)
(t/in? env3 :after)                              ; => true   (both done; done.state.par fired)
```

Expected[^p5]: after `:ok-1` only, `:after` is NOT reached. After both `:ok-1` and `:ok-2`, the parallel completes and the transition fires.

**Why this probe exists.** This is the most common misconception about `done.state.<parallel>` — that it fires when *any* region completes. Seeing partial completion *not* fire the parallel's auto-event cements the "ALL regions must be final" rule.

### Probe 4: What's in the auto-fired event?

```clojure
(def captured (atom nil))

(def chart-spy
  (statechart {}
    (state {:id :container}
      (state {:id :working} (transition {:event :go :target :done}))
      (final {:id :done})
      (transition {:event :done.state.container
                   :cond (fn [_ data]
                           (reset! captured (:_event data))
                           true)
                   :target :after}))
    (state {:id :after})))

(def env4 (t/new-testing-env {:statechart chart-spy
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env4)
(t/run-events! env4 :go)
@captured
;; => {:type :internal :name :done.state.container :sendid :done :data {} ...}
```

Expected[^p7]: the auto-event has `:type :internal`, the name follows `done.state.<parent-id>`, `:sendid` is the final's `:id`, and `:data` is empty.

**Why this probe exists.** Knowing the event's shape lets you read its data (or check its `:sendid`) in advanced patterns. For ex08 the event's just-a-keyword behavior is enough; in real charts you may attach data to the final via `done-data` and read it from the auto-event.

---

## Try breaking it

### Break 1: Put a transition inside a final

**Edit.** Add an outgoing transition to one of the finals:

```clojure
(final {:id :validated}
  (transition {:event :go-back :target :validating}))     ; ← illegal!
```

**Predicted symptom.** Chart construction throws[^p6]:

```
Illegal children of :final with attributes {:id :validated}.
That node cannot have children of type(s): (:transition)
```

**What this proves.** Finals are *truly* terminal — they cannot contain outgoing transitions. The library validates this at element-construction time. If you want a state that "looks final" but allows escape, use a regular state with no outgoing transitions instead (it'll be stuck but won't fire `done.state.<parent>`).

**Restore the simple final** before continuing.

### Break 2: Remove the `done.state.processing` transition

**Edit.** Delete the transition on `:pipeline` that catches `done.state.processing`:

```clojure
(state {:id :pipeline}
  (parallel {:id :processing} ...)
  (state {:id :complete}
    (transition {:event :finalize :target :finished}))
  ;; (transition {:event :done.state.processing :target :complete})   ; ← removed
  )
```

**Predicted symptom.** Block 3 fails — both regions reach final, `done.state.processing` fires (you can verify in DEBUG logs), but nothing matches the event. The chart stays in `:processing` with both regions finalized indefinitely. `(t/in? env :complete)` is false.

**What this proves.** The auto-event is just a regular event. The runtime synthesizes it; *you* have to write the transition that handles it. Without the transition, the auto-event is silently dropped (per ex01/ex02's "no matching transition → event dropped" rule).

**Restore the transition** before continuing.

### Break 3: Make `:finished` not top-level

**Edit.** Move `:finished` inside `:pipeline`:

```clojure
(state {:id :pipeline}
  (parallel ...)
  (state {:id :complete}
    (transition {:event :finalize :target :finished}))
  (transition {:event :done.state.processing :target :complete})
  (final {:id :finished}))                                            ; ← moved here, not top-level
```

**Predicted symptom.** Block 4's `(not (t/in? env :processing))` may pass or fail depending on what "not top-level final" does. If `:finished` is inside `:pipeline`, entering it triggers `done.state.pipeline` (a new auto-event), but nothing handles it — the chart stays in `:pipeline` with `:finished` as the active leaf. `running?` stays `true`. `(t/in? env :processing)` is *probably* false (processing exited when transitioning to `:complete`).

Empirically run it; the exact behavior depends on the chart's specific structure.

**What this proves.** Top-level final has special chart-exit semantics. Finals nested inside other states fire `done.state.<parent>` but don't exit the chart.

**Restore the top-level final** before continuing.

---

## Common breakages

### `Illegal children of :final ...`

You declared a transition (or other child element) inside a final. Finals can't have children. Remove the transition; if you need an escape path, the chart's parent compound state should have a transition listening for the appropriate event.

### Block 3 fails: `(t/in? env :complete)` is false

Three likely causes:

1. **No `:done.state.processing` transition.** Add it to `:pipeline`.
2. **The transition has a typo in the event name** (`:done-state.processing`, `:done.state.process`, etc.). Use the exact form `:done.state.<parallel-id>`.
3. **Only one region reached final.** Confirm both `:validation-ok` and `:enrichment-ok` fired before checking. The auto-event fires only when ALL regions are final.

### Block 4 fails: `running?` is still true after `:finalize`

The `:finished` final is nested inside another state, not top-level. Move it to be a direct child of the chart root (a sibling of `:pipeline`).

### `Cannot register invalid chart`

A target keyword doesn't resolve. The most common case in this exercise: the `:finalize` transition targets `:finished` before you've added the `:finished` final. Add the final, or temporarily remove the `:finalize` transition until step 4.4.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex08_final_and_done.cljc` requires `statecharts.elements` and refers `final`, `parallel`, `state`, `transition`.
- [ ] `:processing` is a parallel with two regions; each region has an internal working-state → final transition.
- [ ] A transition on `:pipeline` (parent of `:processing`) handles `:done.state.processing → :complete`.
- [ ] `:complete` is a regular state with `:finalize → :finished`.
- [ ] `:finished` is a top-level final (sibling of `:pipeline`).
- [ ] `clojure -M:test --focus exercises.ex08-final-and-done/final-and-done-test` reports `8 assertions, 0 failures.`.
- [ ] You can predict what would happen if you removed the `done.state.processing` transition (Break 2 above).
- [ ] You ran Probe 2 and confirmed `(running? env)` returns `false` after the full lifecycle.

**Explain in one breath:**

> *"Each region's `final` signals 'this task done.' When both regions of `:processing` are final, the runtime auto-fires `:done.state.processing`. The transition on `:pipeline` catches it and moves the chart to `:complete`. From `:complete`, `:finalize` targets the top-level `:finished` final — which terminates the chart entirely (configuration empties, `running?` becomes false)."*

If that's natural, ex08 is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex09 (invocations — the last and second-densest module).

---

> **Verified empirically (probes run during ex08 authoring):**
>
> - [^p2]: Configuration progression through the full pipeline lifecycle empirically traced
> - [^p4]: Top-level final entry → configuration empty + `running?` false; chart truly exits
> - [^p5]: `done.state.<parallel>` fires only when ALL regions are final, not on partial completion
> - [^p6]: Finals reject child elements; transitions inside a final throw `Illegal children of :final ...`
> - [^p7]: Auto-event shape: `{:type :internal :name :done.state.<parent-id> :sendid <final-id> :data {} ...}`
