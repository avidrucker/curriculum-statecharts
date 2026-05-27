# Runbook — ex05: Multi-Target Transitions

The operational pass. By the end you'll have edited `src/exercises/ex05_multi_target.cljc` to define a media-player chart with a multi-target `:reset` and seen all four assertions go green.

**Estimated time:** 15–25 minutes if you've completed ex04. The chart is structurally ex04 + one transition; the new mechanic is the `:target` vector.

---

## Prerequisites

- You've completed ex04 (parallel regions). The whole `:playback` / `:volume` chart structure should be familiar.
- You've read [`tutorial.md`](./tutorial.md) for ex05. The runbook references multi-target, the vector form of `:target`, the default-initial fallback, and the same-region footgun without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex05_multi_target.cljc`. ~45 lines. Three things to notice:

1. **The docstring** introduces multi-target via the vector form (`:target [:stopped :unmuted]`) and notes that "without multi-target, a transition into the parallel would just enter each region's default initial state." Hold this nuance — and re-read tutorial Step 2 if it isn't clear.
2. **The TODO comment** also says: "The `:reset` transition should be on the `:player` parallel itself (or on a parent), NOT on an individual region." Following this places the transition correctly.
3. **The `deftest`** has two `testing` blocks with 4 `is` assertions total. The first block drives the chart into `:playing :muted`; the second fires `:reset` and verifies both `:stopped` and `:unmuted` are active.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex05-multi-target/multi-target-test --watch
```

Summary line:

```
1 tests, 4 assertions, 4 failures.
```

Keep the watch terminal open.

---

## Step 3 — Add the elements require

The exercise stub doesn't import the elements. Add:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [parallel state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. Still `4 failures` — expected.

---

## Step 4 — Build the chart, two edits

### 4.1 — Copy the ex04 chart structure (no `:reset` yet)

Replace `;; YOUR CODE HERE` with the ex04 solution structure:

```clojure
(def media-player-with-reset
  (statechart {}
    (parallel {:id :player}
      (state {:id :playback}
        (state {:id :stopped}
          (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}
          (transition {:event :play :target :playing})
          (transition {:event :stop :target :stopped})))
      (state {:id :volume}
        (state {:id :unmuted}
          (transition {:event :toggle-mute :target :muted}))
        (state {:id :muted}
          (transition {:event :toggle-mute :target :unmuted}))))))
```

Save.

Expected — **2 of 4 assertions pass**:

```
1 tests, 4 assertions, 2 failures.
```

The first block ("get into playing + muted state") passes — the ex04 transitions handle `:play` and `:toggle-mute`. The second block ("`:reset` targets both `:stopped` and `:unmuted`") fails: `:reset` has no handler, so the chart stays in `:playing :muted`.

### 4.2 — Add the multi-target `:reset` transition

Add as a direct child of `(parallel {:id :player} ...)`, *before* the regions (or after — placement doesn't affect behavior):

```clojure
(parallel {:id :player}
  (transition {:event :reset :target [:stopped :unmuted]})       ; ← new
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

Save.

Expected — **all 4 assertions pass**:

```
1 tests, 4 assertions, 0 failures.
```

When `:reset` fires while the chart is in `:playing :muted`:

1. The transition matches on the `:player` parallel (which is always active while inside the parallel).
2. The chart exits the parallel (exit-the-whole-parallel rule from ex04).
3. The chart re-enters `:player`, with `:stopped` chosen for `:playback` (from the vector target) and `:unmuted` chosen for `:volume` (from the vector target).
4. Configuration: `#{:player :playback :stopped :volume :unmuted}` — same as initial state.

You've solved ex05.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect and load:

```clojure
(require '[exercises.ex05-multi-target :as ex05] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: The single-target alternative

**Scenario.** You suspect (per the tutorial) that single-target `:target :stopped` would also work. Verify it.

```clojure
;; Replace the :reset transition's target temporarily:
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [parallel state transition]])

(def player-single-target
  (statechart {}
    (parallel {:id :player}
      (transition {:event :reset :target :stopped})       ; single keyword, no vector
      (state {:id :playback}
        (state {:id :stopped} (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}))
      (state {:id :volume}
        (state {:id :unmuted} (transition {:event :toggle-mute :target :muted}))
        (state {:id :muted}   (transition {:event :toggle-mute :target :unmuted}))))))

(def env (t/new-testing-env {:statechart player-single-target
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env :play :toggle-mute)
(t/in? env :playing)         ; → true
(t/in? env :muted)           ; → true
(t/run-events! env :reset)
(t/in? env :stopped)         ; → true
(t/in? env :unmuted)         ; → true   (← also true!)
```

Expected: even with single-target `:target :stopped`, both `:stopped` and `:unmuted` are active after `:reset`. The chart re-enters `:player`, the multi-target only specified `:playback`'s leaf, and `:volume` defaulted to its document-order initial — which is `:unmuted`.

**Why this probe exists.** The exercise's specific test would pass without multi-target. Knowing *why* — that default-initial entry handles the unspecified region — makes the multi-target form a *deliberate choice* (for precise control) rather than a magic incantation.

### Probe 2: Multi-target with a non-default initial

**Scenario.** You want to land the chart in `:stopped + :muted` (not `:unmuted`) with one event.

```clojure
(def player-reset-to-muted
  (statechart {}
    (parallel {:id :player}
      (transition {:event :reset :target [:stopped :muted]})        ; targets :muted, not :unmuted
      (state {:id :playback}
        (state {:id :stopped} (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}))
      (state {:id :volume}
        (state {:id :unmuted} (transition {:event :toggle-mute :target :muted}))
        (state {:id :muted}   (transition {:event :toggle-mute :target :unmuted}))))))

(def env2 (t/new-testing-env {:statechart player-reset-to-muted
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(t/run-events! env2 :reset)
(t/in? env2 :muted)          ; → true    (multi-target forced this; default would be :unmuted)
(t/in? env2 :unmuted)        ; → false
```

Expected: the chart lands in `:muted`. This is where multi-target genuinely shines — when you want a specific non-default initial.

**Why this probe exists.** "Multi-target lets you override default initials" is abstract until you see it work. This probe makes the override concrete.

### Probe 3: The same-region footgun

**Scenario.** You want to see what happens when both targets land in the same region.

```clojure
(def player-broken
  (statechart {}
    (parallel {:id :player}
      (transition {:event :weird-reset :target [:stopped :playing]})     ; both in :playback!
      (state {:id :playback}
        (state {:id :stopped} (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}))
      (state {:id :volume}
        (state {:id :unmuted}))))

(def env3 (t/new-testing-env {:statechart player-broken
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
(t/run-events! env3 :weird-reset)
(t/in? env3 :stopped)     ; → true
(t/in? env3 :playing)     ; → true   (!)
```

Expected: **both `:stopped` AND `:playing` are active simultaneously inside `:playback`** — which is structurally invalid (a compound state should have exactly one active child). The library accepts the configuration anyway.

If you push further, the chart usually self-corrects on the next event whose transition exits one of the two active states:

```clojure
(t/run-events! env3 :play)
(t/in? env3 :stopped)      ; → false  (:stopped's :play transition exited it)
(t/in? env3 :playing)      ; → true   (was already active, still is)
```

So the broken window is brief — but during it, `t/in?` returns true for *both* states. Any code asserting one without expecting the other will be confused.

**Why this probe exists.** Seeing the invalid configuration with your own eyes makes the "every multi-target target must be in a different region" rule stick. If the library validated this, the rule would enforce itself — but it doesn't, so you have to.

### Probe 4: Order in the target vector doesn't matter

```clojure
(def player-order
  (statechart {}
    (parallel {:id :player}
      (transition {:event :reset :target [:unmuted :stopped]})    ; reversed order
      (state {:id :playback}
        (state {:id :stopped})
        (state {:id :playing}))
      (state {:id :volume}
        (state {:id :unmuted})
        (state {:id :muted})))))

(def env4 (t/new-testing-env {:statechart player-order
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env4)
(t/run-events! env4 :reset)
(t/in? env4 :stopped)        ; → true
(t/in? env4 :unmuted)        ; → true
```

Expected: same outcome as `[:stopped :unmuted]`. The library figures out which region each target belongs to without caring about declaration order.

**Why this probe exists.** Knowing that order doesn't matter prevents debugging the wrong thing when a multi-target test fails — the issue is elsewhere.

---

## Try breaking it

### Break 1: Use single-target `:target :stopped`

**Edit.** In the `:reset` transition, drop the vector:

```clojure
(transition {:event :reset :target :stopped})
```

**Predicted symptom.** **Tests still pass.** `:unmuted` is the default-document-order initial of `:volume`, so re-entering the parallel lands `:volume` in `:unmuted` regardless of whether the transition specified it. The exercise's specific test doesn't distinguish single-target from multi-target — they're behaviorally equivalent here.

**What this proves.** Multi-target's value is in precision, not in handling multiple regions per se. When defaults align with intent, single-target works. The runbook keeps the multi-target form because it expresses intent more clearly to a reader ("reset to *both* states") — but functionally it's equivalent.

**Restore the vector** before continuing (for intent clarity).

### Break 2: Use the same-region multi-target

**Edit.** Set the target to two states in the same region:

```clojure
(transition {:event :reset :target [:stopped :playing]})
```

**Predicted symptom.** The chart loads and runs. `:reset` fires; the configuration ends up with both `:stopped` AND `:playing` active inside `:playback`. The test's `(t/in? env :stopped)` passes (because `:stopped` is in the configuration), but `(t/in? env :playing)` *also* returns true — the chart is structurally broken (two atomic states active in a compound region). The chart self-corrects on the next event that exits one of them; until then, the configuration is invalid.

**What this proves.** The library trusts you to put each multi-target in a different region. The runtime doesn't validate the constraint, so the failure mode is silent corruption of the configuration. The chart "still works" in a confusing way (one of the two over-active states will be exited next event), but assertions can pass for the wrong reason.

**Restore `[:stopped :unmuted]`** before continuing.

### Break 3: Make `:reset` a region-level transition

**Edit.** Move the `:reset` transition inside `:playback`'s `:playing` state instead of on the parallel:

```clojure
(state {:id :playing}
  (transition {:event :pause :target :paused})
  (transition {:event :stop :target :stopped})
  (transition {:event :reset :target [:stopped :unmuted]}))      ; ← moved here
```

…and remove it from the parallel.

**Predicted symptom.** The first `testing` block's `:reset` (from `:playing :muted`) passes — `:playing` is active and has the transition. But: if the chart were in `:paused :muted` instead, `:reset` would have no handler and fire nothing. The transition is now scoped to "only fire while `:playing`."

For the test as written, it still passes (because the test drives the chart to `:playing` before `:reset`). But the design intent of "reset works from any state" is gone — a regression no test would catch without changing the test.

**What this proves.** Where you put a transition determines when it fires. On the parallel = always active during the parallel. Inside a region's child = only when that child is active. The exercise's intent is the former; the test happens to exercise only the cases where both forms would work.

**Restore the parallel-level placement** before continuing.

---

## Common breakages

### `Cannot register invalid chart`

A target in your `:target` vector doesn't exist as an `:id`. Grep your chart for `:target [...]` and confirm each keyword in the vector matches an `:id` somewhere. Typos here (`:stoppedd`, `:unmued`) are easy.

### Tests fail after `:reset` — `(t/in? env :stopped)` returns false

Two likely causes:

1. **The transition isn't on the parallel.** If it's inside a region, it only fires from that region. Check Step 4.2 — the transition is a direct child of `(parallel {:id :player} ...)`.
2. **The target vector has a typo.** `:target [:stoppd :unmuted]` would fail at chart registration (see above), but a runtime-only error (`:target [:stopped :unmute]`) would let the chart load with the broken transition silently failing.

### `(t/in? env :muted)` is true after `:reset` (wrong region)

The target vector probably says `:target [:stopped :muted]` instead of `:target [:stopped :unmuted]`. Reading bug — re-check the test's expected outcome and the target value.

### Same-region multi-target leads to weird subsequent behavior

You wrote `:target [:stopped :playing]` or similar — both in the same region. The library doesn't catch this; the chart's configuration is now structurally invalid. Fix the target vector to hit different regions.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex05_multi_target.cljc` requires `statecharts.elements` and refers `parallel`, `state`, `transition`.
- [ ] The chart has the same ex04 structure (parallel `:player` with `:playback` and `:volume` regions).
- [ ] A `(transition {:event :reset :target [:stopped :unmuted]})` declared directly on the `parallel` element.
- [ ] `clojure -M:test --focus exercises.ex05-multi-target/multi-target-test` reports `4 assertions, 0 failures.`.
- [ ] You can articulate why `:target :stopped` (single keyword) would also pass the test, and what's special about ex05 that makes it work.
- [ ] You ran Probe 3 (same-region footgun) and saw the invalid configuration.

**Explain in one breath:**

> *"`:reset`'s `:target` vector explicitly names one state per region — `:stopped` in `:playback`, `:unmuted` in `:volume`. The transition on the parallel exits the whole `:player` and re-enters, with each region landing at its explicitly-targeted state. Multi-target overrides default-initial entry; for this test, the defaults happen to align, so single-target would also work."*

If that's natural, ex05 is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex06 (history states — the tool for *preserving* region state across re-entries, the complement to ex05's "reset to a specific state").
