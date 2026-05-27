# Glossary — ex05: Multi-Target Transitions

Terms introduced (or first deeply covered) in ex05. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex04 covered `parallel`, `region`, multi-element configuration, and the exit-the-whole-parallel rule. This file adds what ex05 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### multi-target transition  *(criticality: 8)*

A `transition` whose `:target` value is a *vector* of state IDs:

```clojure
(transition {:event :reset :target [:stopped :unmuted]})
```

When the transition fires, the chart enters *all* the named states simultaneously. Typically used inside parallels where you want to specify the active leaf in each region precisely (overriding the default initials).

A multi-target transition is structurally a regular `transition` — the only difference from previous exercises is the value type of `:target`.

### `:target` vector form  *(criticality: 8)*

The library accepts `:target` as either a single keyword or a vector of keywords. Internally, single keywords are normalized to one-element vectors (`(if (keyword? target) [target] target)` in `elements.cljc`'s `transition` definition).

| Form | Internal representation |
| --- | --- |
| `:target :stopped` | `[:stopped]` |
| `:target [:stopped]` | `[:stopped]` (unchanged) |
| `:target [:stopped :unmuted]` | `[:stopped :unmuted]` (unchanged) |

There is no special "multi-target" element type. Everything routes through the same `transition` element.

### default-initial fallback  *(criticality: 7)*

When a transition enters a parallel (or any compound state) without specifying *every* leaf, the runtime falls back to each unspecified region's **default initial state** — declared via `:initial` on the region, or the first child in document order if no `:initial` is given.

This is why the exercise's test can be solved with `:target :stopped` (single keyword): re-entering `:player` with only `:stopped` named for `:playback`, the runtime defaults `:volume` to its first child — which is `:unmuted`. Both `(t/in? env :stopped)` and `(t/in? env :unmuted)` end up true.

Multi-target's purpose is to **override the default-initial fallback** in regions where the default isn't what you want.

### same-region multi-target (footgun)  *(criticality: 6)*

The library doesn't validate that the elements of a `:target` vector are in *different* regions. You can write `:target [:stopped :playing]` where both targets are atomic states inside `:playback`. The chart loads; the transition fires; the configuration ends up with **both** `:stopped` and `:playing` active inside `:playback` — which is structurally invalid (a compound state should have exactly one active child).

Empirically: post-event configuration includes both. The library doesn't enforce uniqueness. Subsequent event processing may produce undefined behavior.

Defense: each element of a `:target` vector must be in a different region (or otherwise non-conflicting in the chart's state tree). The library trusts you.

### multi-target on the parallel element  *(criticality: 6)*

A multi-target transition is typically declared as a direct child of the `parallel` element (not inside a region's child), per ex04 Step 7's "transitions on the parallel itself" pattern. This makes the transition fire regardless of which atomic leaves each region currently holds:

```clojure
(parallel {:id :player}
  (transition {:event :reset :target [:stopped :unmuted]})    ; parallel-level
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

If the transition were instead inside one region's atomic state (e.g., `:playing`), it'd only fire from there. Parallel-level placement is the design intent for "this event resets the whole chart from any configuration."

### precision-vs-defaults  *(criticality: 4)*

The pedagogical bottom line of ex05: multi-target is a **precision tool**, not a "multi-region tool." For any chart where the default initials happen to align with what you want, single-target suffices. Multi-target shines when you want a *specific non-default* initial in some region.

A real-world heuristic: if a chart's `:reset`-like transition reads more clearly as `:target [:state-a :state-b]` than `:target :state-a` (even when both work), use the vector — it documents intent.

### target normalization (internal)  *(criticality: 2)*

The `transition` element's source (`com.fulcrologic.statecharts.elements`) handles target normalization at construction time:

```clojure
(let [t (if (keyword? target) [target] target) ...]
  ...)
```

After normalization, the chart's internal representation always stores `:target` as a vector. The library's runtime works exclusively with the vector form; user-facing `:target` accepts either form purely as input ergonomics.

Trivia for now — but useful to know if you're ever introspecting a `pprint`'d chart and see `:target [:foo]` where you wrote `:target :foo`. The library converted it.
