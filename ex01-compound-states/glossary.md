# Glossary — ex01: Compound States & Configuration

Terms introduced (or first deeply covered) in ex01. Cross-cutting vocabulary — [[statechart]], [[state]], [[transition]], [[event]], [[configuration]], [[document-order]] — lives in the top-level [`glossary.md`](../glossary.md); this file only covers what ex01 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `statechart` factory  *(criticality: 9)*

The function call that produces a chart definition:

```clojure
(statechart <chart-options-map>
  <element>
  <element>
  ...)
```

Imported from `com.fulcrologic.statecharts.chart`. The first argument is the **chart options map** (almost always `{}` in the early exercises). The remaining arguments are top-level [[state]] elements (and, later, `parallel`, `final`, `history`, etc.).

The return value is a Clojure map. It does *not* execute anything by itself — handing it to a runtime (in ex01, the [[testing-env]]) is what makes it run.

### `state` element  *(criticality: 9)*

The function from `com.fulcrologic.statecharts.elements` that declares a state node:

```clojure
(state {:id :red} <child-elements>...)
```

The options map *must* include `:id` (a unique keyword within the chart). The body is a sequence of child elements — child states, transitions, on-entry / on-exit actions. For ex01, every state has a body containing exactly one [[transition]] (or none, for whichever state lacks an outgoing transition).

This is a different namespace from the `statechart` factory — both need to be required separately. The exercise stub deliberately omits the elements require so the learner adds it themselves.

### `transition` element  *(criticality: 9)*

The function from `com.fulcrologic.statecharts.elements` that declares a transition:

```clojure
(transition {:event :next :target :green})
```

Two keys appear in ex01:

- **`:event`** — the keyword event name to listen for. (ex02 covers patterns and wildcards.)
- **`:target`** — the `:id` of the state to enter when the transition fires.

The transition is always declared *inside* the source [[state]] element. The "from" is implicit (the enclosing state); the "to" is named via `:target`.

ex03 will add `:cond` (a guard); ex05 will add multi-target via `:target [...]`; ex07 will add `:type :internal`. For ex01, just `:event` and `:target`.

### chart-options map  *(criticality: 4)*

The first argument to `statechart`:

```clojure
(statechart {} ...)
;;          ^
```

For ex01 it's empty. Later exercises will populate it with chart-level configuration (data model setup, naming, hooks). Worth knowing the slot exists — you won't reach for it in ex01.

### `:id`  *(criticality: 8)*

The keyword identifier for a state. Used by [[transition]]s as the value of `:target`. Must be unique within the chart.

By convention, IDs are simple keywords (`:red`), not namespaced (`:traffic/red`). The library allows either, but every exercise in this curriculum uses simple keywords.

### chart root (`:ROOT`)  *(criticality: 7)*

The auto-generated container node that holds the top-level [[state]] elements of a chart. You don't declare it — the `statechart` factory creates it for you with `:id :ROOT` and `:node-type :statechart` — but it *is* a real node in the chart data structure. If you `pprint` the chart, you'll see `:id :ROOT` at the top.

The root plays the role of a [[compound-state]] for the purposes of [[document-order]] — its first child (in source order) becomes the initial state unless an `:initial` is set on the chart options. This is why ex01 doesn't need a `(state {:id :traffic-light} …)` wrapping the three colors: the auto-generated `:ROOT` *is* that wrapper. See [tutorial Step 2](./tutorial.md).

One subtle but important behavior: **`:ROOT` is not in the [[configuration]] set**. After `(t/start! env)`, the configuration is `#{:red}` — not `#{:ROOT :red}`. `(t/in? env :ROOT)` returns `false`. The chart root is the container; the configuration tracks the *contents*.

The library also auto-injects an `initial` element into the chart's `:children` (named something like `:initial25839` — IDs vary per session) pointing at the first state in document order. This is library bookkeeping; you don't write it, you don't reference it, and it doesn't appear in the configuration.

### `t/start!`  *(criticality: 7)*

The function from `com.fulcrologic.statecharts.testing` that initializes a [[testing-env]] and runs the chart's initial transitions:

```clojure
(t/start! env)
```

Before `start!`, the chart isn't in any state — `(t/in? env :red)` returns falsy. After `start!`, the chart is in its initial configuration (for ex01, `:red`). `start!` also runs any on-entry actions on the entered states (none in ex01; ex03 onward).

Calling `start!` twice on the same env *works* — it resets the chart back to the initial configuration. But the **fresh-env pattern** (build a new env via `t/new-testing-env`) is the idiom this curriculum uses for "start a new scenario," because it's clearer to a reader than relying on the reset behavior. See [gotchas #6](./gotchas.md) for the empirical confirmation and the recommended idiom.

### `t/run-events!`  *(criticality: 8)*

The function from `com.fulcrologic.statecharts.testing` that fires one or more events at a running env:

```clojure
(t/run-events! env :next)
(t/run-events! env :event-1 :event-2 :event-3)   ; multiple in sequence
```

Each event is processed in turn:

1. The runtime checks transitions on the currently active state(s) for a match.
2. If a match is found (and any [[guard]] passes), the transition fires — the source state exits, the target enters.
3. The chart settles; the next event is then processed.

If no transition matches, the event is silently dropped. The chart's configuration is unchanged.

### configuration check via `t/in?`  *(criticality: 7)*

The predicate from `com.fulcrologic.statecharts.testing` that asks "is this state ID a member of the current [[configuration]] set?":

```clojure
(t/in? env :red)   ; → true if :red is in the configuration set; false otherwise
```

Internally, `t/in?` is a simple `(contains? configuration-set state-name)`. No hierarchy walk.

For ex01 with three peer atomic states, the configuration after `(t/start! env)` is exactly `#{:red}` — a one-element set containing just the active atomic state. After a `:next` event, it's `#{:green}`. The auto-generated [[chart-root]] (`:ROOT`) is *not* in the set.

Two confusing-on-first-encounter results worth knowing in ex01:

- **`(t/in? env :ROOT) → false`** — the chart root is the container, not a state in the configuration. This stays true in every exercise.
- **`(t/in? env :traffic-light) → false`** — no state in the chart has that ID. The exercise never declared one; the chart root is auto-named `:ROOT`, not `:traffic-light`.

`t/in?`'s set-membership behavior becomes more useful in ex04 (parallel) and ex06 (history), where the configuration set contains multiple state IDs at once.

### "the chart is data" — practical implication  *(criticality: 5)*

The fact that the `statechart` factory returns plain Clojure data — not a class instance, not a process — has three practical consequences worth keeping in mind:

1. **You can `pprint` it.** Useful when debugging chart construction.
2. **You can `def` it once and re-use it.** Different `testing-env`s can wrap the same chart definition; each env has its own configuration and event queue.
3. **You can generate charts programmatically.** Higher-order functions that return chart elements work the same as hand-written elements — this is what enables the convenience macros in ex03c.

You won't lean on this directly in ex01, but the mental model it sets up pays off from ex03 onward.
