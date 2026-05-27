# Glossary — ex04: Parallel Regions

Terms introduced (or first deeply covered) in ex04. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex01–ex03c covered `state`, `transition`, `configuration`, `document-order`, the testing API. This file adds what ex04 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `parallel` (element)  *(criticality: 9)*

The element from `com.fulcrologic.statecharts.elements` that declares an **AND-composition** of regions:

```clojure
(parallel {:id :player}
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

Distinguishing features vs. `state` (compound):

- A compound state has **exactly one** child active at a time. A `parallel` has **all** children active.
- A compound state's initial child is picked by `:initial` or document order. A `parallel`'s initial children are *all* of them — every region activates on entry.
- A compound state's transitions fire based on the active child + ancestor walk. A `parallel`'s transitions are *broadcast* — every region with a matching transition fires.

Common error mode: writing `(state {:id :player} ...)` when you meant `(parallel {:id :player} ...)`. The chart compiles fine; the test fails because only one region is active. See [gotchas.md #1](./gotchas.md).

### region  *(criticality: 8)*

Each immediate child of a `parallel` element is a **region**. Regions are typically compound states; they have their own children, their own initial-state-by-document-order rule, their own transitions.

The defining property: regions are *independent*. Each region's transitions act on its own state; muting in `:volume` doesn't affect `:playback`. Independence emerges structurally — regions only respond to events they declare transitions for. The library doesn't "enforce" independence; it just doesn't couple regions unless you write transitions that cross them.

A region's *own* children are not regions — only the immediate children of `parallel` carry that title. A region's children are normal compound-state children (one active at a time).

### multi-element configuration  *(criticality: 9)*

ex01 established the [[configuration]] as a set. ex02 added a second element (compound parent + atomic child = 2 elements). ex04 is where the set genuinely grows wide.

For the media-player chart, after `(t/start! env)`:

```
#{:player :playback :stopped :volume :unmuted}
```

Five elements: the parallel, both regions, both atomic leaves. **One atomic leaf per region**, plus every ancestor in the active-state chain (excluding `:ROOT`).

`(t/in? env <id>)` is still `(contains? configuration <id>)` — no new logic. The set just has more members. A correct chart with `N` parallel regions will have at minimum `N+1` elements in the configuration (parallel + one atomic per region); often more (compound regions add one element each).

### event broadcast  *(criticality: 8)*

When an event arrives at a parallel chart, the runtime visits **every region** and asks "do you have a matching enabled transition?" Every region that does fires its transition. The regions are processed in the same event-handling cycle; the results all show up in the post-event configuration simultaneously.

Practical implication: declaring a transition for the same event in multiple regions causes all of them to fire on each occurrence. This is sometimes the design (e.g., a `:cancel` event that resets multiple regions); often a footgun (e.g., a `:play` event accidentally also matched in the `:volume` region).

In the exercise's chart, only one region handles each event:

- `:play`, `:pause`, `:stop` — `:playback` only.
- `:toggle-mute` — `:volume` only.

The chart's independence is preserved by this division.

### exit-the-whole-parallel rule  *(criticality: 8)*

When a transition's `:target` is outside the parallel element it was declared in, **the entire parallel exits** — every region, the parallel itself — and the chart re-enters at the target.

```clojure
(parallel {:id :par}
  (state {:id :a}
    (state {:id :a-1}
      (transition {:event :leave :target :outside}))))    ; targets a state outside :par
(state {:id :outside})

;; Before :leave: configuration includes :par and at least one atomic leaf per region.
;; After :leave: configuration is #{:outside}.
```

This is SCXML's behavior, driven by the "LCA" (Least Common Ancestor) algorithm: the transition's exit set is all states inside the LCA of source and target that are currently active. For a cross-parallel transition, the LCA is *above* the parallel, so the parallel's descendants are in the exit set — all regions exit.

To preserve a region while moving another, the transition's `:target` must stay *inside* the parallel. ex06's history states give you tools to re-enter a parallel later while restoring each region's last-active leaf.

### transitions on the `parallel` element itself  *(criticality: 5)*

You can declare transitions as direct children of a `parallel` (alongside the regions, not inside them):

```clojure
(parallel {:id :player}
  (transition {:event :hard-reset :target :off})        ; on the parallel
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

These transitions fire whenever the parallel is active (its ID is in the configuration). Useful for "machine-wide" events that should apply regardless of which atomic leaves each region happens to be in.

When such a transition fires and its target is outside the parallel, it triggers the exit-the-whole-parallel rule.

### region independence (structural, not declarative)  *(criticality: 7)*

The library has no `:independent? true` flag on parallel children. **Independence is the natural result of not declaring cross-region transitions.** If `:volume` region's states only handle `:toggle-mute`, and `:playback` region's states only handle `:play`/`:pause`/`:stop`, the regions evolve without interference.

The corollary: if you accidentally declare a `:play` transition inside the `:volume` region (say, "muting on play"), the regions are now *coupled*. The chart compiles; the behavior is just different.

### atomic state (revisited)  *(criticality: 4)*

ex01 introduced "atomic state" as a state with no children. In a flat chart, the configuration always contains exactly one atomic state. In a parallel chart, the configuration contains *one atomic state per region* — for `N` regions, the configuration has at least `N` atomic leaves.

The "leaf" terminology comes from chart-tree thinking: parallel children are compound branches; each region's active descendant chain ends in an atomic leaf.

### cross-region transition (advanced; rare)  *(criticality: 3)*

A transition whose source is in one region and whose target is in another region of the same parallel. **Possible** in the library; **rarely advisable**.

Empirical behavior (probed in ex04 authoring): the target's region activates the target state, but **the source state stays in the configuration**. For a transition from `:ra-1` (in region `:ra`) to `:rb-2` (in region `:rb`), the post-event configuration is `#{:par :ra :ra-1 :rb :rb-2}` — both `:ra-1` (the un-exited source) *and* `:rb-2` (the new target) are active simultaneously. This is opposite to the clean exit/enter behavior of in-region transitions.

The SCXML-spec reason: the LCA (Least Common Ancestor) of source and target is the parallel itself, so the "exit set" computation includes the target's region but not the source's. The source's region remains in its prior configuration; only the target's region is reshaped.

If you need to coordinate two regions on a single event, the more readable pattern is: declare the event handler in *both* regions explicitly (event broadcast does the rest). Cross-region transitions read as "weird wiring" to anyone reviewing the chart later. The exercise's chart contains no cross-region transitions.
