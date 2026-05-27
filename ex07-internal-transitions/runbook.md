# Runbook — ex07: Internal vs External Transitions

The operational pass. By the end you'll have edited `src/exercises/ex07_internal_transitions.cljc` to define an editor chart with internal transitions and seen all eleven assertions go green.

**Estimated time:** 25–40 minutes if you've completed ex01–ex06. The chart is structurally small but the mechanic (on-entry side effects + internal/external distinction) is the densest single concept since parallel.

---

## Prerequisites

- You've completed ex01–ex06. Comfortable with compound states, on-entry / on-exit, scripts, and the ancestor walk.
- You've read [`tutorial.md`](./tutorial.md) for ex07. The runbook references "external," "internal," the "exit set," and the narrow applicability of `:type :internal` without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex07_internal_transitions.cljc`. ~78 lines. Four things to notice:

1. **The two top-level atoms** (`load-count`, `save-count`) and the two side-effecting functions (`load-doc!`, `save-doc!`). The test counts how many times each runs. The chart's on-entry will call `load-doc!`; on-exit will call `save-doc!`.
2. **The docstring + TODO comment** specify the chart's design: transitions on `:editor` (the parent compound), with `:type :internal` on the two within-compound switches (`:edit`, `:preview`) and external on the exit-the-editor `:done`.
3. **The `(ns …)` form** requires `clojure.test`, `statecharts.chart`, and `statecharts.testing` — but not the elements namespace. You'll add it, with these refers: `on-entry`, `on-exit`, `script`, `state`, `transition`.
4. **The `deftest`** has 4 `testing` blocks containing **11 `is` assertions** total. The longest blocks have 3 assertions each (state + load-count + save-count).

The test resets the atoms at the start (`(reset! load-count 0)` / `(reset! save-count 0)`) — so each test run starts clean.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex07-internal-transitions/internal-transitions-test --watch
```

Summary line:

```
1 tests, 11 assertions, 8 failures.
```

(3 of the 11 pass by accident: every `(= 0 @save-count)` assertion is true because nothing has fired yet, so `save-count` is still 0. The 8 failures are everything else.)

Keep the watch terminal open.

---

## Step 3 — Add the elements require

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [on-entry on-exit script state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. Still `8 failures`.

---

## Step 4 — Build the chart, layer by layer

Four edits.

### 4.1 — Skeleton: `:editor` compound with `:viewing` child + `:closed` sibling

Replace `;; YOUR CODE HERE`:

```clojure
(def editor-chart
  (statechart {}
    (state {:id :editor}
      (state {:id :viewing}))
    (state {:id :closed})))
```

Save.

Expected — **5 of 11 pass**:

```
1 tests, 11 assertions, 6 failures.
```

What now passes:

- Block 1's `(t/in? env :viewing)` — `:editor` is the first top-level state; `:viewing` is its first child by document order.
- Block 3's `(t/in? env :viewing)` — the chart never *left* `:viewing` (no transitions exist), so it's still there after the test fires `:edit` and `:preview` (both silently dropped).
- All three `(= 0 @save-count)` assertions — nothing has fired any script.

What still fails:

- Block 1's `(= 1 @load-count)` — no on-entry exists yet, so load-count is still 0.
- Block 2's `(t/in? env :editing)` — no `:edit` transition; chart stays in `:viewing`.
- Block 2's and block 3's `(= 1 @load-count)` — still 0.
- Block 4's `(t/in? env :closed)` and `(= 1 @save-count)` — no `:done` transition.

### 4.2 — Add on-entry / on-exit + the first internal transition

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (on-exit  {} (script {:expr save-doc!}))
  (transition {:event :edit :target :editing :type :internal})
  (state {:id :viewing})
  (state {:id :editing}))
```

Save.

Expected — **8 of 11 pass**:

```
1 tests, 11 assertions, 3 failures.
```

What's new (compared to 4.1):

- Block 1's `(= 1 @load-count)` ✓ — `:editor`'s on-entry fired on start, incrementing the counter.
- Block 2's `(t/in? env :editing)` ✓ — `:edit` matched the new internal transition; chart switched to `:editing`.
- Block 2's and block 3's `(= 1 @load-count)` ✓ — internal transition didn't re-fire on-entry; counter still 1.

What still fails:

- Block 3's `(t/in? env :viewing)` ✗ — `:preview` has no transition yet, chart stays in `:editing`, not `:viewing`. (Note this *passed* in step 4.1 because the chart was still in `:viewing` from never having moved; now it has moved, so a stale "passes by accident" assertion fails.)
- Block 4's `(t/in? env :closed)` and `(= 1 @save-count)` — no `:done` transition.

### 4.3 — Add the `:preview` internal transition

```clojure
(transition {:event :edit    :target :editing :type :internal})
(transition {:event :preview :target :viewing :type :internal})    ; ← new
```

Save.

Expected — **9 of 11 pass**:

```
1 tests, 11 assertions, 2 failures.
```

Block 3 now fully passes — `:preview` switches back to `:viewing` via another internal transition. Both load-count and save-count assertions in block 3 also pass (still 1 and 0 respectively; internal didn't re-fire anything).

Block 4 still fails — there's no `:done` transition yet.

### 4.4 — Add the `:done` external transition

```clojure
(transition {:event :edit    :target :editing :type :internal})
(transition {:event :preview :target :viewing :type :internal})
(transition {:event :done    :target :closed})    ; ← new (default :type :external)
```

Save.

Expected — **all 11 pass**:

```
1 tests, 11 assertions, 0 failures.
```

When `:done` fires:

- It's external (default), targets `:closed` (outside `:editor`).
- The exit set includes `:editor` and its current descendant.
- `:editor`'s on-exit fires → `save-doc!` runs → `save-count` becomes 1.
- The chart enters `:closed`.

Block 4 now asserts `(t/in? env :closed)` (true) and `(= 1 @save-count)` (true) — both pass[^p1].

You've solved ex07.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect and load:

```clojure
(require '[exercises.ex07-internal-transitions :as ex07] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: Verify the counters across the full cycle

```clojure
(reset! ex07/load-count 0)
(reset! ex07/save-count 0)

(def env (t/new-testing-env {:statechart ex07/editor-chart
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
[@ex07/load-count @ex07/save-count]
;; => [1 0]   (on-entry fired once)

(t/run-events! env :edit)
[@ex07/load-count @ex07/save-count]
;; => [1 0]   (internal transition — :editor wasn't re-entered)

(t/run-events! env :preview)
[@ex07/load-count @ex07/save-count]
;; => [1 0]   (still 1 0)

(t/run-events! env :done)
[@ex07/load-count @ex07/save-count]
;; => [1 1]   (external transition — :editor exited, save-doc! fired)
```

Expected[^p1]: load-count stays at 1 throughout the lifecycle; save-count goes from 0 to 1 only at `:done`. Internal transitions don't touch on-entry/on-exit.

**Why this probe exists.** Watching the counters update (or not) is the most direct way to internalize what `:type :internal` does. The numbers are the spec.

### Probe 2: Compare with external transitions

```clojure
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [on-entry on-exit script state transition]])

(def entries-ext (atom 0))
(def exits-ext (atom 0))

(def chart-external
  (statechart {}
    (state {:id :editor}
      (on-entry {} (script {:expr (fn [_ _] (swap! entries-ext inc) [])}))
      (on-exit  {} (script {:expr (fn [_ _] (swap! exits-ext inc) [])}))
      (transition {:event :edit    :target :editing})        ; default external
      (transition {:event :preview :target :viewing})        ; default external
      (transition {:event :done    :target :closed})
      (state {:id :viewing})
      (state {:id :editing}))
    (state {:id :closed})))

(reset! entries-ext 0)
(reset! exits-ext 0)
(def env2 (t/new-testing-env {:statechart chart-external
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
[@entries-ext @exits-ext]                            ; => [1 0]

(t/run-events! env2 :edit)
[@entries-ext @exits-ext]                            ; => [2 1]  (external re-fires both!)

(t/run-events! env2 :preview)
[@entries-ext @exits-ext]                            ; => [3 2]

(t/run-events! env2 :done)
[@entries-ext @exits-ext]                            ; => [3 3]
```

Expected[^p2]: external transitions on the compound parent re-fire on-entry/on-exit every time. After three events: `entries=3, exits=3`. Compare to Probe 1's `entries=1, exits=1`.

**Why this probe exists.** Seeing the contrast — same chart structure, only the `:type` changed, vastly different side-effect counts — makes the "internal saves cycles" framing concrete.

### Probe 3: Internal on a non-descendant target — silently ignored

```clojure
(def entries-x (atom 0))
(def exits-x (atom 0))

(def chart-internal-outside
  (statechart {}
    (state {:id :editor}
      (on-entry {} (script {:expr (fn [_ _] (swap! entries-x inc) [])}))
      (on-exit  {} (script {:expr (fn [_ _] (swap! exits-x inc) [])}))
      ;; :type :internal, but :closed is OUTSIDE :editor
      (transition {:event :go :target :closed :type :internal})
      (state {:id :viewing}))
    (state {:id :closed})))

(reset! entries-x 0)
(reset! exits-x 0)
(def env3 (t/new-testing-env {:statechart chart-internal-outside
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
[@entries-x @exits-x]                                ; => [1 0]

(t/run-events! env3 :go)
[@entries-x @exits-x]                                ; => [1 1]   (:editor still exited!)
(t/in? env3 :closed)                                 ; => true
```

Expected[^p4]: `:type :internal` is silently ignored when the target isn't a descendant of the source. The transition exits `:editor` anyway. No error, no warning — the annotation is functionally a no-op.

**Why this probe exists.** Knowing that the library doesn't validate the "source compound + descendant target" precondition is the diagnostic clue when "I marked it internal, why does the parent still exit?" The answer is: the target isn't a descendant.

### Probe 4: Internal self-target still re-enters

```clojure
(def entries-self (atom 0))
(def exits-self (atom 0))

(def chart-self
  (statechart {}
    (state {:id :s}
      (on-entry {} (script {:expr (fn [_ _] (swap! entries-self inc) [])}))
      (on-exit  {} (script {:expr (fn [_ _] (swap! exits-self inc) [])}))
      (transition {:event :ping :target :s :type :internal}))))

(reset! entries-self 0)
(reset! exits-self 0)
(def env4 (t/new-testing-env {:statechart chart-self
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env4)
[@entries-self @exits-self]                          ; => [1 0]

(t/run-events! env4 :ping)
[@entries-self @exits-self]                          ; => [2 1]   (re-entered!)
```

Expected[^p6]: even with `:type :internal`, a self-target transition exits and re-enters the state. The annotation has no effect when source == target. Use this knowledge: don't write `:type :internal` on self-loops expecting it to skip on-entry/on-exit.

**Why this probe exists.** Self-loops are common (e.g., counter increment, retry events). Knowing that `:type :internal` doesn't help here is critical — if you need to update data on a self-event without re-firing entry/exit, use a *targetless* transition (ex03b) instead.

---

## Try breaking it

### Break 1: Drop `:type :internal` from `:edit`

**Edit.** Remove the `:type` key:

```clojure
(transition {:event :edit :target :editing})           ; default external
```

**Predicted symptom.** Block 2's `(= 1 @load-count)` fails — the external transition exits `:editor` and re-enters; on-entry re-fires; load-count becomes 2. The block-2 message "load should NOT re-run" is the test runner's confirmation of this exact bug.

**What this proves.** Default external + transition declared on the compound parent + target inside = the parent re-enters on every transition. Internal is what stops this.

**Restore `:type :internal`** before continuing.

### Break 2: Add `:type :internal` to `:done`

**Edit.** Add internal to the done transition:

```clojure
(transition {:event :done :target :closed :type :internal})
```

**Predicted symptom.** **Test still passes.** `:closed` is outside `:editor`; the internal annotation is silently ignored for non-descendant targets (Probe 3). The exit set still includes `:editor`; on-exit still fires; save-count still becomes 1.

**What this proves.** `:type :internal` is a no-op when the target isn't a descendant. The library doesn't validate this precondition. The "right" mental model: use `:type :internal` precisely when you don't want on-entry/on-exit to re-fire on the source compound. Adding it to other transitions is harmless but conveys wrong intent.

**Restore the original `:done` transition** (default external) before continuing.

### Break 3: Move the `:edit` transition into `:viewing`

**Edit.** Pull the `:edit` transition out of `:editor` and put it inside `:viewing`:

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (on-exit  {} (script {:expr save-doc!}))
  (transition {:event :preview :target :viewing :type :internal})
  (transition {:event :done    :target :closed})
  (state {:id :viewing}
    (transition {:event :edit :target :editing :type :internal}))   ; ← now here
  (state {:id :editing}))
```

**Predicted symptom.** Tests *still* pass. The LCA of `:viewing` and `:editing` is `:editor`; even with external (the default), the exit set wouldn't include `:editor`. So `:type :internal` here is redundant.

To make `:edit` actually fail block 2 with a non-internal transition, you'd need `:type :external` AND the transition declared on `:editor` (not `:viewing`). That's the only combination that re-fires the parent's entry/exit.

**What this proves.** Where the transition is declared matters as much as `:type`. Declarations on a child state already keep the parent intact; the `:type :internal` annotation is only meaningful on the parent itself.

**Restore the transition to `:editor`** before continuing.

---

## Common breakages

### Block 1 fails: `(= 1 @load-count)` is 0

You forgot the `on-entry` declaration on `:editor`. Add `(on-entry {} (script {:expr load-doc!}))`.

### Block 2 or 3 fails: load-count is 2 or 3 instead of 1

The within-compound transitions are external (or have no `:type :internal`). External transitions on the compound parent re-fire on-entry on every event. Add `:type :internal` to both `:edit` and `:preview`.

### Block 4 fails: `(= 1 @save-count)` is 0

The `:done` transition is missing or it has `:type :internal` *and* an in-compound target. In the exercise's design, `:done` targets `:closed` (outside) and should be external (default). Remove any `:type :internal` from `:done`.

### Block 4 fails: `(t/in? env :closed)` is false

`:closed` isn't declared, or it's declared inside `:editor` instead of as a top-level state. Confirm `:closed` is a sibling of `:editor` in the chart definition.

### Script doesn't reset between tests

The atoms are top-level. If the test suite runs multiple tests in sequence and your chart uses these atoms, the counts carry over. The test's `(reset! load-count 0)` / `(reset! save-count 0)` handles this — make sure they run before your `(t/start! env)`.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex07_internal_transitions.cljc` requires `statecharts.elements` and refers `on-entry`, `on-exit`, `script`, `state`, `transition`.
- [ ] `:editor` has `on-entry` and `on-exit` blocks containing scripts that call `load-doc!` and `save-doc!`.
- [ ] `:edit` and `:preview` transitions are declared on `:editor`, target descendants, and have `:type :internal`.
- [ ] `:done` transition is declared on `:editor`, targets `:closed`, and is `:type :external` (or has no `:type` — default).
- [ ] `clojure -M:test --focus exercises.ex07-internal-transitions/internal-transitions-test` reports `11 assertions, 0 failures.`.
- [ ] You can predict what would happen if you removed `:type :internal` from `:edit` (load-count goes to 2).
- [ ] You ran Probe 4 (internal self-target) and saw the on-entry re-fire despite the annotation.

**Explain in one breath:**

> *"On `:edit` and `:preview`, internal transitions on `:editor` keep the compound parent in the configuration — its on-entry doesn't re-fire, so `load-doc!` runs once. On `:done`, the external transition (default) targets outside `:editor`, exits the compound, fires on-exit, `save-doc!` runs once. Internal is only meaningful when source is compound and target is a descendant."*

If that's natural, ex07 is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex08 (final states + done.state).

---

> **Verified empirically (probes run during ex07 authoring):**
>
> - [^p1]: `:type :internal` with source=compound, target=descendant — `entries=1, exits=1` across full cycle (only `:done` triggers parent exit)
> - [^p2]: `:type :external` (default) — `entries=3, exits=3` for the same sequence; parent re-fires on every transition
> - [^p4]: `:type :internal` with target outside source's descendant chain is silently ignored; behaves as external
> - [^p6]: `:type :internal` self-target still exits and re-enters
