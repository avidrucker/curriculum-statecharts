# Glossary — ex03b: Data Model Operations & Event Data

Terms introduced (or first deeply covered) in ex03b. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex03's per-exercise glossary covers `:cond`, guard signature, `:_event`, and `op/assign`'s descriptor shape. This file builds on both.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `on-entry`  *(criticality: 9)*

An element from `com.fulcrologic.statecharts.elements`. Wraps executable content that runs when its parent state is **entered**:

```clojure
(state {:id :active}
  (on-entry {}
    (script {:expr (fn [_ _] [(ops/assign :count 0)])}))
  ...)
```

The `{}` is an attribute map (almost always empty). The body is a sequence of executable elements — typically `script`, sometimes `log` or `assign`. The runtime executes them in document order each time the state is entered.

**Important: "each time," not "once."** Every entry — including re-entries via self-targeting transitions, or returning from a child via a normal transition — re-runs the on-entry. If the on-entry resets data, the data resets every time. This is sometimes what you want; sometimes a footgun.

Counterpart: `on-exit`, which fires on state exit. Out of scope for ex03b.

### `script`  *(criticality: 8)*

An element from `com.fulcrologic.statecharts.elements`. Holds an expression the execution model evaluates:

```clojure
(script {:expr (fn [env data] [(ops/assign :count 0)])})
```

The `:expr` is a function in the default (lambda) execution model. Signature: `(fn [env data] -> ops-vector)`. The runtime calls the expression, expects a *vector of operation descriptors*, and applies them to the data model.

Other expression forms — values, vars, symbols, special map shapes — may be accepted by the execution model. See [`ask_tony.md` Q9](../ask_tony.md) and ex03 issue 2 for the open question.

`script` can also load code via `:src` (per its docstring), but no exercise in this curriculum uses that path.

### the vector-of-ops contract  *(criticality: 9)*

The script (and any `:expr` returning operations) must return a **vector** — even when it has only one op or no ops:

| Return | Behavior |
| --- | --- |
| `[op1 op2 ...]` | Applied in order. |
| `[op]` | Applied. |
| `[]` | Nothing applied. (Reasonable — script had nothing to do.) |
| `nil` | Nothing applied. |
| `op` (a single op map, not in a vector) | **Silently ignored.** Data model unchanged, no error. |

The single-op-without-wrapping case is the most common ex03b bug. The error mode is silent — your test fails for what looks like the wrong reason (data is missing) and the runtime doesn't help diagnose. See [gotchas.md #1](./gotchas.md).

### targetless transition  *(criticality: 8)*

A `transition` element with no `:target` key (or `:target nil`):

```clojure
(transition {:event :increment}     ; no :target
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
```

When the event arrives:

1. The transition's body runs (the script's `:expr` is evaluated and any returned ops applied).
2. The chart's configuration is **unchanged** — no state is exited, no state is entered.

This is how charts modify their data model in response to events without "moving anywhere." A `transition + target = same-state` is *not* the same thing — that's a real exit/re-entry cycle that re-fires on-entry/on-exit. Targetless means "no movement at all."

### `handle`  *(criticality: 7)*

A function from `com.fulcrologic.statecharts.convenience`. Pure sugar for `(transition + script)`:

```clojure
(handle :increment
  (fn [_ data] [(ops/assign :count (inc (:count data)))]))
```

is equivalent to:

```clojure
(transition {:event :increment}
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
```

Both produce the same chart structure (modulo auto-generated IDs, which differ — so `=` comparison of charts built with the two forms returns `false`). Use `handle` for readability; fall back to the verbose form when you need more control (e.g., a transition with a guard, body containing multiple elements, etc.).

**Stability caveat.** The convenience namespace's docstring says "ALPHA. NOT API STABLE." `handle` may rename, change arity, or behave differently in future library versions. For long-lived production charts, the verbose form is the safer bet.

### the convenience namespace  *(criticality: 5)*

`com.fulcrologic.statecharts.convenience` — a sibling of the elements namespace that exports sugar functions and macros. ex03b uses `handle`; later exercises may use `on`, `choice`, `send-after`, `assign-on`, the `choice` macro variant, etc. The namespace's own docstring:

> ALPHA. NOT API STABLE.
>
> This namespace includes functions and macros that emit statechart elements, but have a more concise notation for common cases that are normally a bit verbose.

In other words: it's sugar. Convenient, but the maintainer reserves the right to break it. Use the elements namespace as your stability anchor; reach for convenience for readability gains, knowing you might have to update your charts on library upgrades.

### `t/data`  *(criticality: 7)*

A test-helper function from `com.fulcrologic.statecharts.testing`. Returns the current data of the active data model:

```clojure
(t/data env)
;; => {:count 5}    (or nil if the chart hasn't started)
```

Companion to `t/in?` (which queries the configuration). Useful in test assertions:

```clojure
(is (= 0 (:count (t/data env))))
```

Before `(t/start! env)`, `t/data` returns `nil`. After `start!` (and any on-entry scripts have run), it returns whatever map the data model holds.

`t/data` is a test affordance. Production code that needs the data model uses the runtime's protocol-based data-model API directly; you won't call `t/data` outside test contexts.

### event-payload access (`(-> data :_event :data <key>)`)  *(criticality: 8)*

The canonical path for reading the current event's payload inside a `:expr` or `:cond`:

```clojure
(fn [_ data]
  (let [v (-> data :_event :data :value)]
    [(ops/assign :count (or v 0))]))
```

Decomposing the path:

| Position | What it is |
| --- | --- |
| `data` | The function's second arg — session data + injected `:_event`. |
| `:_event` | Runtime-injected key holding the current event map. |
| `:data` | The event map's payload key (set when you call `(t/run-events! env {:name :foo :data {...}})`). |
| `:value` (or whatever) | Your application's key inside the payload. |

The `:data` key inside `:_event` is the payload that the event-firer chose to attach. For events fired as bare keywords (`(t/run-events! env :foo)`), `:data` is `{}` — no payload.

A reasonable defensive pattern: `(or (-> data :_event :data :value) <fallback>)` to handle the case where the event didn't carry the expected field.

### operation descriptor (revisited from ex03)  *(criticality: 6)*

A map that describes a data-model mutation, produced by functions in `com.fulcrologic.statecharts.data-model.operations`:

| Function | Returns | Purpose |
| --- | --- | --- |
| `ops/assign` | `{:op :assign :data {<k> <v>}}` | Write key-value pairs. |
| `ops/delete` | `{:op :delete :paths [<path>]}` | Remove keys from the data model. |
| `ops/set-map-ops` | `[{:op :assign :data <m>}]` | Wrap an existing map into an op vector. |

ex03 introduced `ops/assign` in the context of test setup (`goto-configuration!`). ex03b reuses it in the production context (on-entry scripts and handle bodies). The descriptor itself is the same map either way; what differs is who interprets it (testing helper vs. the chart's runtime).

### alias convention: `op` vs `ops`  *(criticality: 2)*

Cosmetic but worth knowing. The namespace `com.fulcrologic.statecharts.data-model.operations` is aliased two different ways in different exercise files:

- ex03 uses `:as op` (singular)
- ex03b uses `:as ops` (plural)

Both refer to the same namespace. The functions are identical. The exercise stubs and solutions are not consistent; pick whichever reads better in your own code (the plural `ops` matches the namespace's pluralization and reads slightly more naturally as `(ops/assign ...)`, `(ops/delete ...)`).
