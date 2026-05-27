# Glossary — ex03: Guards (Conditions) & Document Order

Terms introduced (or first deeply covered) in ex03. Cross-cutting vocabulary — [[statechart]], [[state]], [[transition]], [[event]], [[configuration]], [[document-order]], [[data-model]], [[guard]], [[action]], [[op/assign]] — lives in the top-level [`glossary.md`](../glossary.md). This file covers what ex03 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `:cond` (guard)  *(criticality: 9)*

The key on a [[transition]] that holds the **guard function** — a predicate the runtime calls to decide whether the transition is enabled when its `:event` matches.

```clojure
(transition {:event :submit
             :cond (fn [env data] (= (:password data) "secret"))
             :target :authenticated})
```

If `:cond` returns truthy, the transition is eligible to fire. If falsy, the transition is skipped and the runtime moves to the next matching transition in [[document-order]]. If `:cond` is *absent*, the transition is unconditionally enabled (treat as "always returns true").

Per the `transition` element's docstring, `:cond` is an "expression that must be true for this transition to be enabled. See execution model." In ex03 the expression is always a function literal `(fn [env data] ...)`; the library's execution model supports other forms (see [`ask_tony.md`](../ask_tony.md) — exact details deferred).

### guard function (signature)  *(criticality: 9)*

The function attached to `:cond`. Fixed signature: **`(fn [env data] -> boolean)`**.

- **`env`** — the runtime environment map. Internal library keys (`::sc/working-memory-store`, `::sc/data-model`, `::sc/event-queue`, `::sc/processor`, `::sc/statechart`, etc.). Rarely read directly in ex03.
- **`data`** — the [[session-data]] map, plus a special `:_event` key holding the current event.

**Truthy/falsy, not boolean-strict.** Returning the string `"false"`, a non-empty collection, or any value other than literal `false`/`nil` counts as truthy.

The runtime invokes the guard exactly **once per transition per event**. It's not called for transitions on states the chart isn't in (efficiency: the ancestor walk only consults active states' transitions).

### session data  *(criticality: 8)*

The chart's per-session, mutable storage area. Lives inside the working memory store under `::sc.dm/data-model` (for the default `FlatWorkingMemoryDataModel`). Before anything sets data, it's `nil`. After [[op/assign]] writes to it, it's a map.

`data` (the second argument to guards, on-entry actions, etc.) is *based on* the session data — but it also includes [[:_event]]. So:

```
session data = {:password "secret"}
data passed to guard = {:password "secret"
                        :_event {:type :external, :name :submit, ...}}
```

The session data is **flat** in the default model — top-level keys, no nesting required. You *can* nest by using a vector path with `op/assign`, but ex03 (and most charts) don't need to.

### `:_event`  *(criticality: 8)*

A special key in `data` (not in the session data itself — the runtime injects it) that holds the current event being processed. Shape:

```clojure
{:type :external
 :name :submit
 :data {:any "event payload"}
 :com.fulcrologic.statecharts/event-name :submit}
```

The event's payload is at `(:data :_event)` — note the double-`:data`. The first refers to the function arg called `data`; the second refers to the `:data` key on the event map.

A guard that wants to read event-time information uses `(get-in data [:_event :data :some-key])`. See runbook Probe 3 for the working example.

### the fallback pattern  *(criticality: 7)*

The idiomatic way to express "do A if some condition; otherwise do B" in a chart:

```clojure
(state {:id :login}
  (transition {:event :submit :cond <some-check> :target :authenticated})
  (transition {:event :submit                    :target :rejected}))
```

Two transitions on the same event in the same state. The first carries a `:cond`; the second has no `:cond` (= unconditional). Document order makes the guarded one tried first; the unguarded one becomes the "else branch."

This depends on three rules:
1. [[document-order]] determines which transition is tried first.
2. Missing `:cond` is unconditionally enabled.
3. First match wins (short-circuit).

Break any one of these and the fallback pattern stops working as intended. See runbook "Try breaking it" for the failure modes.

### `t/goto-configuration!`  *(criticality: 6)*

A **test-time-only** helper from `com.fulcrologic.statecharts.testing`. Takes:

```clojure
(t/goto-configuration! env data-ops leaf-states)
```

- `data-ops` — a vector of operation descriptors (typically `[(op/assign :foo "bar")]`).
- `leaf-states` — a set of atomic state IDs naming where the chart should "be."

What it does:

1. Runs the `data-ops` against the data model.
2. Sets the configuration to `(configuration-for-states env leaf-states)` — the leaf states *plus* their proper ancestors (including `:ROOT`).
3. Validates the resulting configuration; throws if it's invalid (e.g., you asked for a leaf state that doesn't exist).

Use it when a test needs the chart in a specific state with specific data, and driving the chart there from `start!` via several `run-events!` calls would be tedious. *Don't* use it in production code; the runtime never accepts arbitrary "place me here" commands at runtime.

There's a notable asymmetry vs `start!`: `goto-configuration!` includes `:ROOT` in the configuration set, where `start!` excludes it. See [gotchas.md #1](./gotchas.md).

### `data-model.operations` namespace  *(criticality: 5)*

The library namespace exposing operation constructors:

| Function | Returns | Purpose |
| --- | --- | --- |
| `op/assign` | `{:op :assign :data {<k> <v>}}` | Write key-value pairs into the data model |
| `op/delete` | `{:op :delete :paths [<path>]}` | Remove keys from the data model |
| `op/set-map-ops` | `[{:op :assign :data <m>}]` | Wrap an existing map into an op vector |

ex03 only uses `op/assign`. ex03b will cover the rest plus the data model implementation in depth.

The "operation as data" pattern is what makes the chart fully data-describable: an entire chart, including its data manipulations, is a Clojure value you can `pprint`, diff, and generate programmatically.

### short-circuit (transition selection)  *(criticality: 6)*

The runtime's transition-selection algorithm stops at the **first** transition (in [[document-order]] within a state, then walking ancestors per ex02) whose `:event` matches *and* whose `:cond` returns truthy. Subsequent transitions are not consulted.

Practical consequences:

- A guard's side effects (don't have them) might or might not run, depending on whether earlier transitions also match.
- Putting expensive guards later (after cheap ones that often pass) is a free optimization — when the cheap guards pass, the expensive ones are skipped.
- The "fallback pattern" only works because of this.

### data model implementation (default)  *(criticality: 3)*

The library ships several data model implementations; the default in the testing environment is `FlatWorkingMemoryDataModel` (in `com.fulcrologic.statecharts.data-model.working-memory-data-model`). It stores session data as a flat map inside the working memory store.

You won't need to think about which data model is in use during ex03. The reason it's worth knowing: when the library docs say "see your data model implementation for the interpretation of path vectors," they're referring to the fact that *different* data models (e.g., a Fulcro-integrated one) might interpret `(op/assign [:a :b] 1)` as "set `:b` under the `:a` scope" vs. "set the path `[:a :b]`." The flat model treats keywords as top-level keys.
