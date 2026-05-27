# Gotchas — ex04: Parallel Regions

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex04. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex03c gotchas still apply. This file adds what `parallel`, multi-element configuration, event broadcast, and the exit-the-whole-parallel rule newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. Using `state` where you meant `parallel`

**What you'll see.** You write `(state {:id :player} (state {:id :playback} ...) (state {:id :volume} ...))`. The chart compiles. `(t/start! env)` succeeds. `(t/in? env :stopped)` returns `true`. `(t/in? env :unmuted)` returns **`false`**. You wonder why one region is "missing."

**What's really happening.** `state` is a *compound* state — exactly one child is active at a time. Your two-region chart is now a one-region chart: the runtime picks `:playback` (first in document order) as the active child of `:player`, enters it, and lands in `:stopped`. `:volume` is *never entered*. The configuration after `start!` is three elements (`:player :playback :stopped`), not five.

There's no error. The chart just behaves like half of what you intended.

**Defense.** When you want multiple regions active simultaneously, the element is `parallel`, not `state`. Memorize the rule: **`state` with multiple children = alternative initials; `parallel` with multiple children = simultaneously-active regions.** When a test failure on a multi-region chart shows one region active and the other not, this is the first thing to check.

*See also:* [tutorial Step 1](./tutorial.md), [runbook "Try breaking it" #1](./runbook.md), [glossary `parallel`](./glossary.md).

---

### 2. `:ROOT` is *still* excluded from the configuration in parallel charts

**What you'll see.** You inspect the configuration of a running parallel chart, expect `:ROOT` to show up (after all, the parallel is "at the root level"), and find it isn't there.

```clojure
(config env)
;; => #{:player :playback :stopped :volume :unmuted}
;; No :ROOT.
```

**What's really happening.** Same rule from ex01 — the auto-generated chart root `:ROOT` is not in the configuration set, even when a `parallel` element is at the top of the chart. The runtime treats `:ROOT` as the container; the parallel below it is a real chart element that *does* appear in the configuration.

**Defense.** When walking the configuration for diagnostic purposes, ignore `:ROOT` — it'll never be there. Use the parallel's `:id` (e.g., `:player`) if you want a "machine is alive" check.

(One exception, still applies: `(t/goto-configuration! ...)` *includes* `:ROOT`. Same asymmetry as ex03. See [ex03/gotchas.md #1](../ex03-guards/gotchas.md).)

---

### 3. The first assertion fails when neither region's first child is what the test expects

**What you'll see.** Your chart structure looks right, but `(t/in? env :stopped)` is `true` and `(t/in? env :unmuted)` is `false`. Or vice versa. Or both wrong, but one of them not what you wanted.

**What's really happening.** When a `parallel` is entered, each region enters its *first child in document order* (unless `:initial` overrides). If `:muted` appears before `:unmuted` inside `(state {:id :volume} ...)`, then `:muted` is the initial, not `:unmuted`.

**Defense.** Two options:

1. **Use document order:** put the intended initial state first in each region.

   ```clojure
   (state {:id :volume}
     (state {:id :unmuted}                  ; ← first; will be initial
       (transition {:event :toggle-mute :target :muted}))
     (state {:id :muted}
       (transition {:event :toggle-mute :target :unmuted})))
   ```

2. **Use explicit `:initial`:** declare it on the region:

   ```clojure
   (state {:id :volume :initial :unmuted}
     (state {:id :muted} ...)
     (state {:id :unmuted} ...))
   ```

Either works. Document-order-first is more idiomatic and matches the test's intuition.

---

### 4. Two regions both declaring a transition for the same event

**What you'll see.** You're refactoring the media player. You add a `:reset` event that should clear playback (`→ :stopped`). You also add `:reset` in `:volume` to clear mute (`→ :unmuted`). All four states inside `:playback` now have `:reset → :stopped`; both states inside `:volume` have `:reset → :unmuted`.

Now you fire `:reset` while the chart is playing and muted. Both regions update. The chart enters `:stopped` and `:unmuted` simultaneously. You think you've designed an elegant chart.

Months later, a colleague adds a `:reset` event that should *only* reset playback (for a "stop without unmuting" feature). They add it to one region; the chart's *other* region also fires its `:reset`, because that transition was never removed. Now `:reset` resets both regions even though only one was intended.

**What's really happening.** Event broadcast means every region with a matching transition fires. Multiple regions handling the same event = all of them update. This is sometimes desirable (one event, multiple coordinated effects); often surprising (the regions weren't supposed to be coupled).

**Defense.** When adding an event to a parallel chart, ask: "which regions should respond?" Add the transition only to those. Document the design choice with a comment:

```clojure
;; :reset clears playback AND volume — by design.
;; If you want to reset only one region, add a more specific event (e.g., :reset-playback).
```

The discipline matters more in long-lived charts than in exercises.

*See also:* [tutorial Step 4 + 5](./tutorial.md), [runbook "Try breaking it" #3](./runbook.md).

---

### 5. A transition with `:target` outside the parallel collapses the whole parallel

**What you'll see.** You write what looks like a normal transition in one of your parallel's leaves, targeting a state at the chart root:

```clojure
(state {:id :playing}
  (transition {:event :emergency-stop :target :emergency}))
;; ... and :emergency is declared outside the :player parallel
```

When `:emergency-stop` fires, you expect `:playing` to exit and the chart to enter `:emergency`. **You don't expect `:volume` to also exit.** But it does — the configuration after `:emergency-stop` is `#{:emergency}`. The whole `:player` parallel is gone. Whatever `:volume` was tracking is lost.

**What's really happening.** SCXML's LCA (Least Common Ancestor) rule: the transition's exit set is all currently-active states inside the LCA. For a transition from inside `:player` targeting a state outside `:player`, the LCA is the chart root — so the exit set is `:player` and everything in it (both regions, every active leaf). The whole parallel exits.

**Defense.** When you write a transition out of a parallel, **decide what should happen to the other region(s):**

- **Reset is fine?** No problem — the parallel re-enters with all regions back to their initial states next time it activates.
- **Need to preserve regions' state?** Use a `history` state (ex06). History remembers each region's last-active leaf and restores them on re-entry.
- **Need to update only one region?** The `:target` must be *inside* the parallel (typically a sibling of the source's enclosing region, or within the same region).

*See also:* [tutorial Step 6](./tutorial.md), [runbook Probe 5](./runbook.md), [glossary `exit-the-whole-parallel rule`](./glossary.md).

---

### 6. Cross-region transitions: source state stays in the configuration

**What you'll see.** You write a transition in one region's child targeting a state in the *other* region's child:

```clojure
(parallel {:id :par}
  (state {:id :ra}
    (state {:id :ra-1}
      (transition {:event :cross :target :rb-2}))    ; targets :rb-2 in :rb
    (state {:id :ra-2}))
  (state {:id :rb}
    (state {:id :rb-1})
    (state {:id :rb-2})))
```

You fire `:cross`. You expect `:ra-1` to exit (it's the transition's source) and `:rb-2` to enter. **What actually happens (verified empirically):**

```
before :cross  →  #{:par :ra :ra-1 :rb :rb-1}
after  :cross  →  #{:par :ra :ra-1 :rb :rb-2}    ; ← :ra-1 is STILL active!
```

`:ra-1` stays in the configuration. `:rb`'s active leaf switched from `:rb-1` to `:rb-2` as expected. But the source region (`:ra`) wasn't re-initialized — `:ra-1` is still "active" alongside the now-active `:rb-2`.

**Contrast with an in-region transition:**

```clojure
(state {:id :ra-1}
  (transition {:event :within :target :ra-2}))     ; targets :ra-2 within :ra
```

```
before :within  →  #{:par :ra :ra-1 :rb :rb-1}
after  :within  →  #{:par :ra :ra-2 :rb :rb-1}    ; ← :ra-1 exits cleanly
```

The in-region case exits the source state and enters the target — the clean behavior you'd expect from ex01–ex03.

**What's really happening.** SCXML's exit-set rules for transitions are computed from the LCA (Least Common Ancestor) of source and target. For an in-region transition, the LCA is *inside* the region — so only the source state exits and the target enters. For a cross-region transition, the LCA is the parallel itself — and the runtime's interpretation of "exit set" doesn't necessarily include the source's region (since the target enters one region and the source's region is structurally still "supposed to be" active in the parallel context). The result: an ambiguous-looking configuration with the source's leaf still present.

**Defense.** **Don't write cross-region transitions in your own charts.** They're legal but their behavior is subtle and inconsistent across SCXML implementations. If you need to coordinate two regions on a single event, declare the event handler in *both* regions explicitly — event broadcast does the rest, and the result is predictable:

```clojure
;; Good: each region handles :cross on its own terms
(state {:id :ra}
  (state {:id :ra-1}
    (transition {:event :cross :target :ra-2}))
  (state {:id :ra-2}))
(state {:id :rb}
  (state {:id :rb-1}
    (transition {:event :cross :target :rb-2}))
  (state {:id :rb-2}))
```

This is also why the exercise's chart has no cross-region transitions: it'd be teaching a pattern that produces surprising configurations.

*See also:* [glossary `cross-region transition (advanced; rare)`](./glossary.md).

---

### 7. Forgetting that a parallel's *own* `:id` is in the configuration

**What you'll see.** You write a test that wants to know "is the chart still alive?" or "is the user in the player flow?" You reach for `(t/in? env :playback)` — sure, but only because `:playback` happens to be a region you know is active. Eventually `(t/in? env :playback)` returns `false` because the parallel exited.

But `:player` (the parallel itself) is in the configuration as long as the parallel is active. `(t/in? env :player)` is the cleanest "is the player machine running?" predicate.

**What's really happening.** The configuration includes the parallel's `:id`, every region's `:id`, and every active atomic leaf. The parallel's ID is the most stable signal of "the parallel is still active" — regions' IDs are too, but using the parallel's ID expresses intent better.

**Defense.** When you want to know "is the user in this multi-region machine?", check the parallel's ID, not a region's. Reserve region-ID checks for "is this specific region active?" (which is *always* true if the parallel is active, but reads as more specific).

---

### 8. Empty regions (no atomic-state children) — degenerate but legal

**What you'll see.** You write `(state {:id :playback})` (no children) as a region inside a parallel:

```clojure
(parallel {:id :player}
  (state {:id :playback})              ; ← empty
  (state {:id :volume}
    (state {:id :unmuted})))
```

The chart loads. After `(t/start! env)`, `(t/in? env :playback)` is `true`, but `(t/in? env :stopped)` is `false` (`:stopped` doesn't exist yet — and even if it did, `:playback` has no children to enter). The configuration is `#{:player :playback :volume :unmuted}` — four elements, missing the atomic leaf for `:playback`.

**What's really happening.** A region with no children is an "atomic region" — it has no internal state to track. The runtime activates it (puts its ID in the configuration) but there's no further descent. Such regions are uncommon in practice; the exercise's intermediate Step 4.1 has this shape briefly, but the final solution doesn't.

**Defense.** When you `pprint` the configuration of a parallel chart and a region's atomic leaf is "missing," check whether the region declared any child states. If not, add one (and any transitions).

---

### 9. Adding a region after the chart is in motion

**What you'll see.** You add a `:third-region` to your `parallel` while the chart is running (`(def media-player (statechart {} ...))` re-evaluated). Old testing envs based on the prior chart still reference the prior definition; new envs see the updated chart. You're surprised when an existing env doesn't pick up the new region.

**What's really happening.** Same lesson as ex01 gotcha #8 — a testing env captures the chart definition at creation time. Re-defining the chart's `var` doesn't update existing envs. To see the new region, build a fresh env with `(t/new-testing-env ...)`.

**Defense.** When iterating on a parallel chart at the REPL, build a fresh env after every `def` re-eval. The fresh-env-per-scenario discipline applies double here because parallel charts are easier to misread when you forget you're looking at a stale env.

---

### 10. `(t/data env)` and parallel: still flat (no per-region data partitioning)

**What you'll see.** You expect each region to have its own data model (one for `:playback`, one for `:volume`). You write `(handle :play (fn [_ data] [(ops/assign :playback-state :playing)]))` and `(handle :toggle-mute (fn [_ data] [(ops/assign :volume-state :muted)]))`. You read `(t/data env)` and find one flat map with both keys.

**What's really happening.** The default `FlatWorkingMemoryDataModel` (which the curriculum uses throughout) is one global map per chart, not per-region. All regions write to the same data model. You can use namespacing or nested maps to partition the data conceptually, but the runtime doesn't enforce a partition.

**Defense.** If you want per-region data isolation, name your keys clearly:

```clojure
(handle :play (fn [_ _] [(ops/assign :playback/state :playing)]))
(handle :toggle-mute (fn [_ _] [(ops/assign :volume/state :muted)]))
```

The data model becomes `{:playback/state :playing :volume/state :muted}`. Naming convention does the partitioning the library doesn't.
