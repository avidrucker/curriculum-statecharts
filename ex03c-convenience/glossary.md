# Glossary — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

Terms introduced (or first deeply covered) in ex03c. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex03b's per-exercise glossary covered `handle`, `on-entry`, `script`, the vector-of-ops contract, `:_event` path, and `t/data`. This file builds on both.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `on` (convenience helper)  *(criticality: 8)*

Function from `com.fulcrologic.statecharts.convenience`. Pure sugar for a `transition` element:

```clojure
(on :submit :checking)
;; expands to
(transition {:event :submit :target :checking})
```

Signature: `(on event target & actions)`. The extra `actions` (variadic body) become children of the emitted transition — useful when you want a simple state-change transition that also runs a script or logs. ex03c doesn't use the variadic form.

`on`'s sole purpose is brevity. It is *not* a different transition concept — the runtime can't distinguish an `on`-emitted transition from a hand-written one.

### `choice` (convenience helper)  *(criticality: 9)*

Function from `com.fulcrologic.statecharts.convenience`. Emits a **state element** containing **eventless transitions** that act as a `cond`-style dispatch on data:

```clojure
(choice {:id :auth-check}
  pred-1? :target-1
  pred-2? :target-2
  :else   :fallback-target)
```

Expands to:

```clojure
(state {:id :auth-check :diagram/prototype :choice}
  (transition {:cond pred-1? :target :target-1})
  (transition {:cond pred-2? :target :target-2})
  (transition {:target :fallback-target}))            ; :else clause
```

Three things to know:

1. **It's a state**, not a new element type. `:node-type :state`. The `:diagram/prototype :choice` is a hint for chart-rendering tools; the runtime ignores it.
2. **Its transitions are eventless** — they fire on entry, not on an external event.
3. **The `:else` clause is special** — extracted from the clause list regardless of declaration position and appended as the *last* transition (no `:cond`, so it's the unconditional fallback).

### eventless transition  *(criticality: 8)*

A `transition` element with no `:event` key (or `:event nil`). Such transitions are evaluated automatically when their state is entered — and during "settling" cycles between events — gated only by `:cond` (if present).

```clojure
(state {:id :a}
  (transition {:target :b}))           ; eventless, unconditional — fires on entry
```

Critically: **the chart never lingers in a state whose eventless transitions fire successfully.** Upon entering `:a`, the runtime immediately evaluates the eventless transition; it fires; the chart exits `:a` and enters `:b`. From a test's point of view, `(t/in? env :a)` returns `false` because by the time you can check, the chart has moved on.

The exception: if a state's eventless transitions all have `:cond`s that return false, *and* there's no unconditional fallback, the chart **gets stuck** in that state. See `[[stuck-choice-pattern]]`.

### transient state pattern  *(criticality: 6)*

A state the chart enters and immediately exits via eventless transitions, without lingering. **Choice states are the canonical example.** Any state whose only outgoing transitions are eventless is a transient state.

Practical consequences:

- `(t/in? env :transient-id)` returns `false` (almost) always — the chart's "in" the state for too short a time to observe.
- The state's `:id` still appears in the chart's structure and configuration *transiently*, but post-settling it's gone.
- Transient states with on-entry actions still run their on-entry — even though the chart's stay is brief, the actions fire on every entry.

### stuck-in-choice failure mode  *(criticality: 8)*

The empirical fact, verified in this exercise's probes: **a `choice` without `:else` whose predicates all return false leaves the chart stuck in the choice state**. No exception, no warning. The chart sits in the choice with no eligible transition. Subsequent events that don't match a transition somewhere else in the active configuration are silently dropped.

Defense: always include `:else <some-state>` in a `choice` unless you can prove every reachable data-model state is covered by the conditional clauses. The `:else` is the safety net.

### `:_event` in eventless predicates  *(criticality: 5)*

When a choice (or any eventless-transition state) is entered as part of processing an external event, the triggering event is *still* available in `(-> data :_event)`:

```clojure
;; Chart fires :go, lands in a state containing a choice.
;; Choice predicate runs:
(fn [_ data] (-> data :_event :name))    ; → :go     (the triggering event)
```

When the chart enters a choice automatically — e.g., the choice is the chart's initial state — `:_event` is `nil`. Empirically verified in runbook Probe 3.

Practical implication: a choice can dispatch on event payload (via `:_event :data`), not just on session data. This is how to collapse multiple `handle`s into one choice when the dispatch logic depends on which event arrived.

### the convenience namespace (deeper coverage)  *(criticality: 5)*

ex03b mentioned `com.fulcrologic.statecharts.convenience` and its "ALPHA. NOT API STABLE." status. ex03c uses three of its helpers — `on`, `handle`, `choice` — and surfaces two more in the source that the curriculum doesn't drill into yet:

| Helper | Emits | Purpose |
| --- | --- | --- |
| `on` | `(transition {:event :target ...})` | Simple event-triggered transition |
| `handle` | `(transition {:event ...} (script {:expr ...}))` | Event handler that updates data, no state change |
| `choice` | `(state ... eventless transitions)` | `cond`-style dispatch on entry |
| `assign-on` | `(transition {:event ...} (assign {...}))` | `handle`-like with `assign` element instead of script — not in ex03c |
| `send-after` | A pair of on-entry / on-exit with `send` and `cancel` | Timeout-style delayed event delivery — covered in later exercises |
| `choice` (macro variant) | Same as fn `choice` plus annotation notes for diagram rendering | Stylistic |
| `on` (macro variant) | (none in 1.2.25) | — |

ex03c only requires `on`, `handle`, and `choice`. The macro variants are not used here.

### `:diagram/prototype`  *(criticality: 2)*

A key the `choice` helper adds to the emitted state: `:diagram/prototype :choice`. The runtime *ignores it* — chart execution treats the state like any other. The key is a hint for chart-visualization tools (a TODO in the library, per the convenience namespace's docstring) that want to render choice states with a distinct visual style (e.g., a diamond).

You'll see this key in `pprint`ed chart definitions and can safely ignore it for behavior reasoning. Trivia worth knowing so it doesn't confuse you on first encounter.

### `(t/in? env :choice-id)` — when it might be true  *(criticality: 4)*

For a normally-functioning `choice` state, `(t/in? env :choice-id)` returns `false` post-event-processing — the chart entered and exited within the same processing cycle.

**`(t/in? env :choice-id)` returns `true`** only when the chart is *stuck* in the choice (the [[stuck-choice-pattern]] failure mode). This is sometimes useful as a diagnostic — if a test fails with `(t/in? env :expected-target)` returning false, also check `(t/in? env :the-choice-id)`; if it's `true`, you've hit the no-`:else` trap.
