# Runbook — ex04: Parallel Regions

The operational pass. By the end you'll have edited `src/exercises/ex04_parallel.cljc` to define a media-player chart with two parallel regions and seen all ten assertions go green.

**Estimated time:** 40–60 minutes if you've completed ex03. The chart is structurally bigger than anything you've built (two compound regions inside a parallel, 5+5 states total) and the mental model takes a beat to settle. The tests are easy once the structure is right.

---

## Prerequisites

- You've completed ex01–ex03c. Comfortable with `state`, `transition`, multi-state configurations, document order.
- You've read [`tutorial.md`](./tutorial.md). The runbook references parallel, regions, event broadcast, multi-element configuration, and the exit-the-whole-parallel rule without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex04_parallel.cljc`. ~60 lines. Four things to notice:

1. **The docstring** explicitly calls out "two independent concerns" and lists every event with which region handles it. The independence story is the spine — keep it in mind when wiring transitions.
2. **The TODO comment** sketches the chart shape: `(parallel {:id :player} (state {:id :playback} ...) (state {:id :volume} ...))`. The top-level element is `parallel`, not `state`.
3. **The `(ns …)` form** requires `clojure.test`, `statecharts.chart`, and `statecharts.testing`. It does **not** require `statecharts.elements`. You'll add it — and need to refer `parallel`, `state`, `transition`.
4. **The `deftest`** has five `testing` blocks, each with **two** `is` assertions (one per region). Total: **10 assertions**. The test is verifying after each event that *both* regions are in their expected state.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex04-parallel/parallel-test --watch
```

Summary line:

```
1 tests, 10 assertions, 10 failures.
```

Keep the watch terminal open.

---

## Step 3 — Add the elements require

The exercise stub doesn't import the elements you need. Add the require line:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [parallel state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Three things to refer: `parallel` (new), `state`, `transition`. Save. The runner re-runs and still reports `10 failures` — expected.

---

## Step 4 — Build the chart, region by region

Three edits. We build the skeleton (parallel + each region with only its initial atomic state) first, then fill in `:playback` and `:volume` in turn. The pass count rises monotonically: 4 → 8 → 10.

### 4.1 — The skeleton: parallel + each region with only its initial atomic state

Replace `;; YOUR CODE HERE`:

```clojure
(def media-player
  (statechart {}
    (parallel {:id :player}
      (state {:id :playback}
        (state {:id :stopped}))
      (state {:id :volume}
        (state {:id :unmuted})))))
```

Save.

Expected — **4 of 10 pass**:

```
1 tests, 10 assertions, 6 failures.
```

What passes:

| testing block | which `is` | why |
| --- | --- | --- |
| "starts in :stopped and :unmuted" | both | parallel activates both regions; document-order picks `:stopped` and `:unmuted` |
| "play starts playback, volume unaffected" | `(t/in? env :unmuted)` only | `:volume` has no `:play` transition, so it stays in `:unmuted` |
| "unmute while paused" | `(t/in? env :unmuted)` only | the chart never left `:unmuted` (no transition handler exists yet) |

The other 6 fail because nothing moves anywhere — no transitions are defined.

Configuration at this point: `#{:player :playback :stopped :volume :unmuted}` — five elements.

### 4.2 — Add the full `:playback` region

Replace the placeholder `(state {:id :playback} (state {:id :stopped}))` with the full region:

```clojure
(state {:id :playback}
  (state {:id :stopped}
    (transition {:event :play :target :playing}))
  (state {:id :playing}
    (transition {:event :pause :target :paused})
    (transition {:event :stop :target :stopped}))
  (state {:id :paused}
    (transition {:event :play :target :playing})
    (transition {:event :stop :target :stopped})))
```

(Leave `:volume`'s placeholder as-is.)

Save.

Expected — **8 of 10 pass**:

```
1 tests, 10 assertions, 2 failures.
```

What's new:

- `:play` now moves `:stopped` → `:playing`. Block 2 ("play starts playback, volume unaffected") fully passes — `(t/in? env :playing)` is true *and* `(t/in? env :unmuted)` is still true (the `:volume` region wasn't affected by `:play` — independence at work).
- `:toggle-mute` arrives next but `:volume` doesn't handle it; the chart stays in `:playing` and `:unmuted`. Block 3 ("toggle-mute mutes, playback unaffected") gets the `:playing` check (true) but fails the `:muted` check.
- `:pause` moves `:playing` → `:paused`. Block 4 ("pause pauses, mute unaffected") gets the `:paused` check (true) but again fails the `:muted` check.
- A final `:toggle-mute` is still ignored. Block 5 ("unmute while paused") gets both checks: `(t/in? env :paused)` is true, and `(t/in? env :unmuted)` is *still* true because the chart never muted in the first place.

The two remaining failures are both `(t/in? env :muted)` — once in block 3, once in block 4. `:volume` is still in `:unmuted` because nothing handles `:toggle-mute` yet.

### 4.3 — Add the full `:volume` region

Replace the placeholder `(state {:id :volume} (state {:id :unmuted}))` with:

```clojure
(state {:id :volume}
  (state {:id :unmuted}
    (transition {:event :toggle-mute :target :muted}))
  (state {:id :muted}
    (transition {:event :toggle-mute :target :unmuted})))
```

Save.

Expected — **all 10 assertions pass**:

```
1 tests, 10 assertions, 0 failures.
```

The `:toggle-mute` event now toggles `:unmuted` ↔ `:muted`. Both `:muted` checks in blocks 3 and 4 now pass. Block 5's `(t/in? env :unmuted)` *also* still passes — between block 3's first toggle (mute) and block 5's second toggle (unmute), the region returns to `:unmuted`.

You've solved ex04.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect (`clojure -M:nrepl` then your editor, or plain `clojure`):

```clojure
(require '[exercises.ex04-parallel :as ex04] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: Inspect the configuration set

```clojure
(defn config [env]
  (-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
      :storage deref :test
      (get :com.fulcrologic.statecharts/configuration)))

(def env (t/new-testing-env {:statechart ex04/media-player
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(config env)
;; => #{:player :playback :stopped :volume :unmuted}
```

Expected: five elements. The parallel (`:player`), both regions (`:playback`, `:volume`), both active atomic leaves (`:stopped`, `:unmuted`). **`:ROOT` is not in the set** (same rule as every prior exercise).

**Why this probe exists.** When something in a parallel chart breaks, the first question is "what does the chart think it's in?" The configuration set is the answer. Make this reflexive.

### Probe 2: Verify independence — `:play` doesn't touch `:volume`

```clojure
(t/in? env :unmuted)   ; => true   (before any event)
(t/run-events! env :play)
(t/in? env :playing)   ; => true
(t/in? env :unmuted)   ; => true   (UNCHANGED — :volume not affected)
(config env)
;; => #{:player :playback :playing :volume :unmuted}
```

Expected: `:playback` moved (`:stopped` → `:playing`); `:volume` untouched (`:unmuted` still active). Independence isn't a special library mechanism — it's just that `:unmuted`'s state has no transition for `:play`.

### Probe 3: Verify broadcast — `:toggle-mute` from any playback state

```clojure
;; While in :paused
(t/run-events! env :pause)
(t/in? env :paused) ; => true
(t/run-events! env :toggle-mute)
(t/in? env :paused) ; => true (unchanged — :playback didn't handle :toggle-mute)
(t/in? env :muted)  ; => true (volume responded)
```

Expected: `:toggle-mute` is broadcast to both regions. Only `:volume` has a matching transition, so only `:volume` changes. `:playback` continues in `:paused` because nothing there matches.

### Probe 4: The configuration after a chain of events

```clojure
(def env2 (t/new-testing-env {:statechart ex04/media-player
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(config env2)
;; => #{:player :playback :stopped :volume :unmuted}

(t/run-events! env2 :play :toggle-mute :pause)
(config env2)
;; => #{:player :playback :paused :volume :muted}
;; (or similar — the set order is not stable, but membership is)
```

Expected: after three events, the configuration has changed both atomic leaves (`:stopped` → `:paused`, `:unmuted` → `:muted`) but the parallel + region IDs are unchanged. The chart structure is invariant; only the active leaves shift.

### Probe 5: The exit-the-whole-parallel rule

```clojure
;; Build a chart that has a "shutdown" event escaping the parallel:
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [parallel state transition]])

(def shutdownable-player
  (statechart {}
    (parallel {:id :player}
      (state {:id :playback}
        (state {:id :stopped}
          (transition {:event :shutdown :target :off}))    ; exits the parallel
        (state {:id :playing}
          (transition {:event :shutdown :target :off}))
        (state {:id :paused}
          (transition {:event :shutdown :target :off})))
      (state {:id :volume}
        (state {:id :unmuted})
        (state {:id :muted})))
    (state {:id :off})))    ; outside the parallel

(def env3 (t/new-testing-env {:statechart shutdownable-player
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
(config env3)
;; => #{:player :playback :stopped :volume :unmuted}

(t/run-events! env3 :shutdown)
(config env3)
;; => #{:off}
```

Expected: After `:shutdown`, the configuration is just `#{:off}`. The whole parallel is gone. Even though `:shutdown` was fired from `:stopped` (in `:playback`), the entire `:player` parallel — and both regions — collapsed.

**Why this probe exists.** This is the most counter-intuitive rule in parallel charts. Once you've seen it with your own eyes, you'll never forget that "a transition out of a parallel exits *everything*."

---

## Try breaking it

### Break 1: Use `state` instead of `parallel`

**Edit.** Replace `(parallel {:id :player} ...)` with `(state {:id :player} ...)`:

```clojure
(state {:id :player}                       ; ← was parallel
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

**Predicted symptom.** First assertion of block 1 fails: `(t/in? env :unmuted)` is `false`. The chart loads (it's a valid compound state structure), but `:player` is now a *compound* state — exactly one of its children is active. The runtime picks `:playback` (first in document order) and only enters it. `:volume` (and `:unmuted`) are never active. The configuration is `#{:player :playback :stopped}` — three elements, not five.

**What this proves.** `parallel` vs `state` is the single most consequential element choice in a chart with multiple active regions. Visually they look almost identical; semantically they're as different as it gets.

**Restore `parallel`** before continuing.

### Break 2: A transition out of one region targets the other region's leaf

**Edit.** Change `:playing`'s `:stop` transition to target `:muted` (a state in the *other* region):

```clojure
(state {:id :playing}
  (transition {:event :pause :target :paused})
  (transition {:event :stop :target :muted}))            ; was :stopped; now :muted (in :volume)
```

**Predicted symptom.** Tests pass for `:play` (block 2), but eventual `:stop` behavior is bizarre. Empirically: the chart enters `:muted` (in `:volume`), exits the prior `:playing` (in `:playback`), but `:playback` region's active leaf is no longer well-defined. The configuration might be `#{:player :playback :playing :volume :muted}` (with `:playing` *still* active) or `#{:player :playback :stopped :volume :muted}` (with `:playback` re-initialized) depending on SCXML's cross-region semantics.

**What this proves.** Cross-region transitions are exotic. They work, but the configuration result isn't always intuitive. Avoid them unless you know exactly why you want one. For ex04, every transition's `:target` is in the same region as its source.

**Restore the `:stopped` target** before continuing.

### Break 3: Have both regions handle the same event

**Edit.** Add a `:play` transition inside `:unmuted`:

```clojure
(state {:id :unmuted}
  (transition {:event :toggle-mute :target :muted})
  (transition {:event :play :target :muted}))            ; new — also fires on :play
```

**Predicted symptom.** When `:play` fires, *both* regions transition: `:playback` moves `:stopped` → `:playing` (as before), and `:volume` *also* moves `:unmuted` → `:muted`. After `:play`, the configuration is `#{:player :playback :playing :volume :muted}`. The test's block 2 fails: `(t/in? env :unmuted)` is `false`, because the volume region just moved to `:muted`.

**What this proves.** Event broadcast is real — every region with a matching transition fires. "Region independence" depends entirely on regions not declaring transitions for events from other regions.

**Restore the original `:unmuted`** before continuing.

---

## Common breakages

### `Cannot register invalid chart` when the chart loads

A target keyword doesn't exist as an `:id`. With ten states and lots of transitions to keep straight, typos accumulate. Grep for `:target` references; confirm each matches an `:id` somewhere:

```sh
grep -oE ':target :[a-z-]+' src/exercises/ex04_parallel.cljc | sort -u
grep -oE ':id :[a-z-]+' src/exercises/ex04_parallel.cljc | sort -u
```

Any `:target` keyword not in the `:id` list is a bug.

### First assertion fails: `(t/in? env :stopped)` is false (or `(t/in? env :unmuted)` is false)

Two possibilities:

1. **You used `state` instead of `parallel`.** Only one of `:playback` or `:volume` is active, and probably not both. See Break 1 above.
2. **The atomic states aren't first in their region's document order.** Reorder so `:stopped` is first inside `:playback` and `:unmuted` is first inside `:volume`. (Or declare `:initial :stopped` on `:playback` and `:initial :unmuted` on `:volume`, but document-order-first is more idiomatic.)

### `:play` moves to `:playing` but `(t/in? env :unmuted)` returns false

Either:

1. **You forgot the `parallel` element** (Break 1 again — symptom shows up here too).
2. **You accidentally put a `:play` transition in one of `:volume`'s states** that targets a non-`:unmuted` state. Check `:volume`'s children for a stray transition.

### Tests pass except `:muted` ones — `(t/in? env :muted)` returns false

The `:volume` region's `:toggle-mute` transitions aren't wired up. Check Step 4.4 — both `:unmuted` and `:muted` need a `:toggle-mute` transition targeting the other.

### Toggling mute repeatedly doesn't stay consistent

The two `:toggle-mute` transitions need to target each other:

- `:unmuted`'s `:toggle-mute → :muted`
- `:muted`'s `:toggle-mute → :unmuted`

If both target the same state (e.g., both say `→ :muted`), the chart gets stuck after the first toggle.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex04_parallel.cljc` requires `statecharts.elements` and refers `parallel`, `state`, `transition`.
- [ ] `media-player` has a top-level `(parallel {:id :player} ...)`.
- [ ] `:player`'s children are `(state {:id :playback} ...)` and `(state {:id :volume} ...)` — both compound states.
- [ ] Each region's children are atomic states with the right transitions.
- [ ] `clojure -M:test --focus exercises.ex04-parallel/parallel-test` reports `10 assertions, 0 failures.`.
- [ ] You can predict the configuration set at any moment without running the chart.
- [ ] You ran Probe 5 (exit-the-whole-parallel) and saw `#{:off}` after `:shutdown`.

**Explain in one breath:**

> *"`:player` is a parallel — when entered, both regions activate, each landing in its first-document-order atomic leaf (`:stopped` and `:unmuted`). Events broadcast to every region; only regions with a matching transition fire. `:play` updates `:playback` without touching `:volume`; `:toggle-mute` updates `:volume` without touching `:playback`. Independence emerges from regions only handling their own events."*

If that's natural, ex04 is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex05 (multi-target transitions).
