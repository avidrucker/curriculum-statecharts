# Glossary — cross-cutting terms

Vocabulary that every exercise in this curriculum builds on. Per-exercise glossaries (e.g., [`ex03-guards/glossary.md`](./ex03-guards/glossary.md)) cover terms first introduced in that exercise; this file covers the terms used in *every* exercise.

If you're new to statecharts, skim this top to bottom once before opening ex01 — not to memorize, but to recognize the words when they show up.

---

## The 0–9 criticality scale

Each entry is rated by how badly *not* understanding it will hurt you on later exercises:

| Score | Meaning |
| --- | --- |
| **9** | Blocking — you cannot complete an exercise without it. |
| **7–8** | Critical — the term carries load in every exercise; misunderstanding it costs you hours. |
| **5–6** | Useful — you'll meet it often; rough mental model is enough at first. |
| **3–4** | Background — encountered as vocabulary, doesn't drive decisions. |
| **0–2** | Trivia — worth knowing for context, not for getting unstuck. |

The 9s on this page are the spine. If you can explain "statechart," "state," "event," "transition," and "configuration" in one breath each, every exercise becomes legible.

---

## Core constructs

### statechart  *(criticality: 9)*

The whole machine — a *named, executable description* of how a system behaves over time. It bundles together a set of [[state]]s, the [[transition]]s between them, optional [[data-model]]s, and rules for what happens when [[event]]s arrive.

In Tony Kay's library, you build a statechart with the `statechart` factory:

```clojure
(statechart {opts}
  (state {:id :A} (transition {:event :go :target :B}))
  (state {:id :B}))
```

The returned value is plain Clojure data — a map describing the chart. The map has `:id :ROOT`, `:node-type :statechart`, and a `:children` vector. It does **not** run by itself. To make it run, you hand it to an "environment" (in tests, [[testing-env]]; in apps, a working environment created at startup).

Etymologically descends from David Harel's 1987 paper on statecharts and the W3C [[SCXML]] spec; the library's behavior follows SCXML where the spec is unambiguous. The library also exposes `scxml` as an alias for `statechart`, for readers who prefer the SCXML naming.

### state  *(criticality: 9)*

A named situation the chart can be in. In the library, declared with the `state` element:

```clojure
(state {:id :idle} ...)
```

States come in three flavors that you'll learn to spot:

- **Atomic** — no children. The leaf of the tree.
- **[[compound-state]]** — has child states, exactly one of which is "active" at a time. (When a compound state is active, both *it* and its currently-active child appear in the [[configuration]] set — except for the auto-generated chart root `:ROOT`, which is never in the configuration.)
- **Parallel** — declared with `parallel`, has child states *all* of which are active simultaneously. (Covered in ex04.)

A state is identified by its `:id`. IDs must be unique within the chart.

### transition  *(criticality: 9)*

A directed move from one state to another, triggered by an [[event]]. Declared inside the source state:

```clojure
(state {:id :red}
  (transition {:event :next :target :green}))
```

Transitions can also have a [[guard]] (`:cond`), an [[action]] body, and multiple targets. Without an `:event`, a transition is "eventless" — runs as soon as its conditions are met (advanced; not exercised in the early modules).

### event  *(criticality: 9)*

A signal sent to the chart, usually a keyword. Events are how the outside world drives the chart forward. You fire one in tests with:

```clojure
(t/run-events! env :next)
```

Events can have **data** (a payload, separate from session data). Events can also be **internal** (raised by the chart itself, e.g., the `done.state.<id>` event that fires when a [[compound-state]]'s child reaches a final state — ex08). They can be **namespaced** keywords (`:user/login`); event matching against patterns is the subject of ex02.

### configuration  *(criticality: 8)*

The **set of state IDs the chart is currently active in**, stored as a Clojure set under `::sc/configuration` in the chart's working memory.

For a flat chart (ex01), the configuration is just the active atomic state — e.g., `#{:red}`. Before `t/start!` runs, the configuration is absent (`nil`). After `start!` of the traffic light, it becomes `#{:red}`; after a `:next` event, `#{:green}`.

For a nested chart with explicit compound states (ex04 parallel, ex06 history), the configuration set will contain *multiple* IDs — every concurrently-active atomic state plus any compound-state ancestors the spec marks as "in." The auto-generated chart root (`:ROOT`) is **not** included in the set: `(t/in? env :ROOT)` returns `false` even when the chart is running.

You query the configuration in tests with `t/in?`:

```clojure
(t/in? env :red)         ; => true if :red is a member of the configuration set
```

Internally, `t/in?` is a simple `(contains? configuration-set state-name)` — there's no hierarchy walk, no ancestor traversal. If a state's ID is in the set, you're "in" it; otherwise you're not. The interesting behavior in nested charts (ex04 onward) comes from the runtime *putting* multiple IDs into the set, not from any cleverness in `t/in?`.

### document order  *(criticality: 8)*

The order in which states and transitions appear *as written* in the chart definition. Document order is **load-bearing** in two places:

1. **Initial state selection.** When a [[compound-state]] is entered without an explicit `:initial`, the *first* child state in document order becomes the active child.
2. **Transition selection.** When multiple transitions in the same state match the same event, the *first one in document order* whose [[guard]] passes wins.

This is why ex03's guarded transitions must come *before* the unguarded fallback — same event, two transitions, document order is the tiebreaker.

---

## Working with charts

### compound state  *(criticality: 7)*

A state with child states. Exactly one child is active at a time. Compound states have an *initial* child (either declared with `:initial` or implied by [[document-order]]). Entering a compound state means entering the compound state *and* its initial child (and recursively, that child's initial child).

This recursion is what people mean by "hierarchical state machines" — a state is a node in a tree, not a single bucket.

### parallel state  *(criticality: 6)*

A state whose children are all active simultaneously (covered in detail in ex04). Each child is called a [[region]]. Events sent to a parallel state are delivered to every region.

### region  *(criticality: 5)*

One of the concurrent children of a [[parallel-state]]. Each region is independently a [[compound-state]] (or atomic). The term is mostly used in ex04 onward.

### initial state  *(criticality: 7)*

The state a [[compound-state]] enters by default. Specified either explicitly:

```clojure
(state {:id :traffic-light :initial :red}
  (state {:id :red} ...)
  (state {:id :green} ...))
```

or implicitly via [[document-order]] (the first child wins).

### final state  *(criticality: 5)*

A child state declared with `final` that, when entered, signals "this region is done." Reaching a final state inside a [[compound-state]] raises a `done.state.<parent-id>` internal event, which the parent can transition on. Covered in ex08.

---

## The data model

### data model  *(criticality: 6)*

The per-session storage the chart can read and write. Lives alongside the configuration; gets passed into [[guard]]s and [[action]]s. In the library, the default data model is an in-memory map; for app integration there are pluggable backends.

The data model is **not** the event payload. Events carry their own data; the data model is the chart's *own* state-shaped scratchpad.

### action  *(criticality: 5)*

Code that runs as a side effect of entering a state (`on-entry`), exiting a state (`on-exit`), or taking a [[transition]] (transition body). Actions are how a chart causes effects in the outside world — log a message, send an HTTP request, set a flag in the [[data-model]].

### `op/assign`  *(criticality: 6)*

The convention for writing to the [[data-model]] from inside an action. Imported from `com.fulcrologic.statecharts.data-model.operations`. Used in ex03 onward to set up test data and in ex03b extensively.

```clojure
(op/assign :password "secret")
```

### guard  *(criticality: 7)*

A condition that gates a [[transition]] — only if the guard returns truthy does the transition fire. Declared with `:cond`:

```clojure
(transition {:event :submit :cond check-password :target :authenticated})
```

The guard is a function of `(env, data) → boolean`. Covered in detail in ex03.

---

## The library API surface you'll meet in every exercise

### `com.fulcrologic.statecharts.chart`  *(criticality: 7)*

The namespace that exposes `statechart` — the top-level factory. Always imported in test files as `:refer [statechart]`.

### `com.fulcrologic.statecharts.elements`  *(criticality: 7)*

The namespace exposing the building-block elements: `state`, `transition`, `parallel`, `final`, `history`, `invoke`, plus on-entry / on-exit element wrappers. Imported as `:refer [state transition ...]` per exercise.

### `com.fulcrologic.statecharts.testing` *(usually aliased `t`)*  *(criticality: 9)*

The test-side API. The four functions you'll see in every exercise:

| Function | What it does |
| --- | --- |
| `t/new-testing-env` | Create a fresh chart "environment" (the runtime). Takes `{:statechart ... :mocking-options {...}}`. |
| `t/start!` | Initialize the environment — runs the chart's entry actions and lands it in the initial configuration. |
| `t/run-events!` | Fire one or more events at the chart. Each event is processed; the chart settles before returning. |
| `t/in?` | Predicate — is the named state in the current [[configuration]]? |

These four are enough to drive every chart in the exercise set.

### testing env  *(criticality: 6)*

The runtime container produced by `t/new-testing-env`. Holds the chart definition, the [[data-model]] state, the current [[configuration]], and the event queue. Each test creates a fresh one with `let`.

The `:mocking-options {:run-unmocked? true}` map is the default invocation — it tells the testing harness "if I don't explicitly mock an invocation or side effect, just run it." Worth knowing about; not load-bearing for any exercise.

---

## Background / not-required-reading

### SCXML  *(criticality: 3)*

[State Chart XML](https://www.w3.org/TR/scxml/) — the W3C standard that codifies David Harel's statechart concept into an interchange format. Tony Kay's library follows SCXML semantics where the spec is unambiguous (event-processing order, transition selection rules, parallel-region behavior).

You will never write XML in this curriculum. The reason SCXML matters is that if you ever need to look up "what does the spec say should happen when…", the SCXML document is the canonical reference.

### David Harel  *(criticality: 1)*

The Israeli computer scientist who introduced statecharts in [a 1987 paper](https://www.sciencedirect.com/science/article/pii/0167642387900359). Worth a Wikipedia visit if you're curious about the origins; not relevant to solving exercises.

### Tony Kay  *(criticality: 1)*

Author of Fulcro and of `com.fulcrologic/statecharts`. The talks in [`../fulcro-statecharts-talks/`](../fulcro-statecharts-talks) are the canonical introduction to this library specifically; they double as the lecture-track companion to this curriculum's exercise-track.

---

## Per-exercise vocabulary index

Terms introduced (or first deeply covered) in each module's local glossary, not here:

| Exercise | New terms |
| --- | --- |
| ex02 | event-name matching, event pattern, wildcards in event names |
| ex03 | `:cond` (deep), session data setup via `on-entry` + `op/assign` |
| ex03b | data operations beyond `assign` |
| ex03c | convenience macros (sugar for raw `statechart` element construction) |
| ex04 | `parallel` element, region semantics, event broadcast |
| ex05 | multi-target transitions, transition fanout |
| ex06 | shallow history, deep history, default-history target |
| ex07 | internal transitions, the `:type :internal` flag, no-exit semantics |
| ex08 | `final` state, `done.state.<id>` event |
| ex09 | `invoke` element, `:src`, child charts, `done.invoke.<id>` event |
