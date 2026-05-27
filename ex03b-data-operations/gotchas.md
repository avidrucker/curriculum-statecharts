# Gotchas — ex03b: Data Model Operations & Event Data

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex03b. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01, ex02, and ex03 gotchas still apply. This file adds what data operations + `script` + `handle` + the convenience namespace newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. A script returning a *single op* (not in a vector) silently does nothing

**What you'll see.** You write:

```clojure
(script {:expr (fn [_ _] (ops/assign :count 0))})    ; no brackets
```

The chart starts. No exception. No log warning. `(t/data env)` returns `nil` instead of `{:count 0}`. Every downstream assertion fails because `:count` is missing.

**What's really happening.** The runtime expects `:expr` to return a *vector of operation descriptors*. When it receives a single map (`{:op :assign :data {:count 0}}`), it tries to iterate over the map and feed each key-value pair to the op interpreter. Nothing in the iteration matches the expected op shape, so no ops are applied. The runtime does not warn — it treats this as "the script chose to apply zero ops."

**Defense.** Always wrap. Even when there's exactly one op. The form is:

```clojure
;; CORRECT — even with one op
(script {:expr (fn [_ _] [(ops/assign :count 0)])})

;; CORRECT — multiple ops
(script {:expr (fn [_ _] [(ops/assign :count 0)
                          (ops/assign :history [])])})

;; WRONG — silently does nothing
(script {:expr (fn [_ _] (ops/assign :count 0))})
```

If your data model doesn't update when you expect it to, this is the first thing to check. Make a habit of grepping your charts for `:expr (fn` followed by something that's clearly *not* a vector literal.

*See also:* [tutorial Step 2](./tutorial.md), [runbook Probe 2 and "Try breaking it" #1](./runbook.md).

---

### 2. `on-entry`'s `data` is `nil` on the chart's first entry

**What you'll see.** You write an on-entry script that does `(inc (:count data))`, intending to increment a counter that you "know" was set to `0` somewhere. The chart starts; the script runs; you get an NPE from `(inc nil)`.

**What's really happening.** On-entry scripts run *before* their own ops are applied to the data model. On the chart's very first entry to a state, the data model has not yet been written to — `data` is `nil`. A subsequent on-entry (after the chart has been running) will see whatever's already in the data model.

```clojure
;; FRAGILE — fails on first entry because data is nil
(on-entry {} (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))

;; DEFENSIVE — works on first entry and later
(on-entry {} (script {:expr (fn [_ data] [(ops/assign :count (inc (or (:count data) 0)))])}))

;; SAFEST — pure write, no read at all
(on-entry {} (script {:expr (fn [_ _] [(ops/assign :count 0)])}))
```

**Defense.** In on-entry initialization, prefer pure writes over read-modify-write. If you must read first (e.g., to merge with previous data), `or`-default every read against an explicit fallback.

*See also:* [tutorial Step 1 timing subtlety](./tutorial.md), [runbook "Try breaking it" #2](./runbook.md).

---

### 3. A `transition` with `:target` equal to its own enclosing state is *not* targetless

**What you'll see.** You want a transition that updates data without "going anywhere." You write:

```clojure
(state {:id :active}
  (on-entry {} (script {:expr (fn [_ _] [(ops/assign :count 0)])}))
  (transition {:event :increment :target :active}             ; targeting self
    (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])})))
```

You fire `:increment` once. You expect `:count` to be `1`. Instead, `:count` is *still* `0`.

**What's really happening.** `:target :active` causes the chart to **exit `:active`, then re-enter `:active`** — a real lifecycle event. The exit happens, then the re-entry runs `on-entry`, which resets `:count` to `0`, then the transition's script runs (depending on execution order, this may apply after the reset, leaving `:count` at `1` — or before, leaving it at `0`). Either way, the on-entry's reset is firing on every `:increment`, which is almost never what you want for a counter.

The fix: drop the `:target`. A targetless transition fires its body without exiting/entering the enclosing state — no on-entry/on-exit cycle.

```clojure
(transition {:event :increment}      ; ← no :target
  (script {:expr (fn [_ data] [(ops/assign :count (inc (:count data)))])}))
```

Or use the convenience `handle`, which is exactly this shape:

```clojure
(handle :increment (fn [_ data] [(ops/assign :count (inc (:count data)))]))
```

**Defense.** When you write a transition meant to "stay where I am and update data," **omit `:target` entirely**. Don't write `:target :the-current-state`. Use `handle` from convenience to make the intent unambiguous.

*See also:* [tutorial Step 3](./tutorial.md), [runbook "Try breaking it" #3](./runbook.md).

---

### 4. The convenience namespace is *ALPHA, NOT API STABLE*

**What you'll see.** You rely on `handle` throughout a real chart. A library upgrade changes its arity, renames it to `on-event`, or moves it. Your charts stop loading.

**What's really happening.** The convenience namespace's own docstring says:

> ALPHA. NOT API STABLE.

The library author reserves the right to evolve these sugar functions. The underlying patterns they emit (`transition` + `script`, etc.) are SCXML-aligned and stable; the sugar is not.

**Defense.** Two practices:

1. **For long-lived production charts**, prefer the elements-namespace forms (`transition`, `script`, etc.) directly. They're more verbose but won't break on library upgrades.
2. **For curriculum-following / exploratory code**, `handle` is fine. Just know that when you upgrade the library, scan the convenience namespace's changelog for breaking changes.

The exercises use `handle` because the topic is data operations, not API stability. Real code should make the choice deliberately.

*See also:* [tutorial Step 4 stability warning](./tutorial.md), [glossary `handle`](./glossary.md).

---

### 5. `handle` returns a transition with `:target nil` — that's not an error

**What you'll see.** You `(pprint (handle :go (fn [_ _] [])))` and see something like:

```clojure
{:event :go
 :target nil       ; ← what
 :type :external
 :children [{:node-type :script ...}]}
```

`:target nil` looks like a bug ("the target should be a keyword"). It isn't.

**What's really happening.** A `transition` with no `:target` is targetless — the chart doesn't move. The library represents this internally as `:target nil`. When you read someone else's chart in the REPL and see `:target nil`, that's the explicit representation, not an oversight.

**Defense.** `:target nil` in a printed transition map = targetless transition. Don't try to "fix" it by setting `:target :some-state`.

---

### 6. `:expr` functions with the wrong arity error at runtime, not chart-load

**What you'll see.** You write `(handle :go (fn [data] ...))` — accidentally 1-arg. The chart loads fine. When you fire `:go`, you get `Wrong number of args (2) passed to: ...`.

**What's really happening.** The library doesn't validate `:expr` arities at chart construction. The function is just stored as a value. When the runtime calls it (always with 2 args: `env` and `data`), Clojure raises ArityException.

**Defense.** Always write `:expr` functions with the 2-arg signature, even when you only need `data`. Use `_` or `_env` to name the unused first arg:

```clojure
(fn [_ data] [(ops/assign :count (:count data))])
;; or
(fn [_env data] [(ops/assign :count (:count data))])
```

This is the same arity convention as ex03's guards.

---

### 7. Comparing two charts with `=` returns `false` even when they're structurally equivalent

**What you'll see.** You want to verify `handle` produces the same structure as the verbose form. You write:

```clojure
(= (handle :go my-fn)
   (transition {:event :go} (script {:expr my-fn})))
;; => false
```

**What's really happening.** Every `transition` and `script` element auto-generates an `:id` keyword (`:transition17564`, `:script17563`, etc.). The IDs are generated fresh on each call, so structurally equivalent elements have different IDs and `=` returns false.

**Defense.** Don't use `=` to compare chart structures. If you want to verify equivalence, compare with the IDs stripped (`(dissoc m :id)`), or compare *behaviorally* by running both charts through a test env and checking observable state.

*See also:* [glossary `handle`](./glossary.md) — same point.

---

### 8. `ops/assign` for nested paths: the default data model treats keywords as flat keys

**What you'll see.** You try to set a nested value:

```clojure
(ops/assign [:user :name] "Alice")
```

You expect `data` to become `{:user {:name "Alice"}}`. Instead, you see `{[:user :name] "Alice"}` — or get an error depending on the data model.

**What's really happening.** The library's *default* data model (the `FlatWorkingMemoryDataModel` used by tests and standalone charts) treats every key as a top-level entry. Vector paths are *not* interpreted as `assoc-in`-style nesting in this model. Other data models (notably the Fulcro-integrated one) interpret paths differently — the `op/assign` docstring explicitly says "see your data model implementation for the interpretation."

**Defense.** For ex03b and the curriculum's CLJ-only context, stick to flat keyword keys: `(ops/assign :user-name "Alice")`. If you need nested structure, store a map:

```clojure
(ops/assign :user {:name "Alice" :id 42})
```

When you later move to Fulcro, learn the integrated data model's path semantics separately — they'll be different.

---

### 9. `(t/data env)` after a script's ops have applied — but *which* script's ops?

**What you'll see.** You're debugging. You call `(t/data env)` after `(t/run-events! env :increment)` and see what you expect. But if you call `(t/data env)` from *inside* a script's `:expr`, you can't — `t/data` requires a `testing-env` (the outer test object), and you only have `env` (the runtime infrastructure map) inside the script.

**What's really happening.** Two different "env"s exist:

- The **testing env** (`t/data`, `t/in?`, `t/start!`, `t/run-events!` all take this). Built by `t/new-testing-env`. Has keys like `:env`, `:session-id`, `:statechart`.
- The **runtime env** (the `env` argument inside guards/scripts). The inner runtime infrastructure map. Has keys like `::sc/working-memory-store`, `::sc/data-model`, etc.

You read the data model inside a script via the `data` argument (the script's second arg). Outside the runtime, you use `(t/data testing-env)`. They're not interchangeable.

**Defense.** If you need to log the data model from inside a script: just use `data` (it's the same map `t/data` would return for the same moment). Print it with `(println "DATA:" data)` and the data is right there.

```clojure
(script {:expr (fn [env data]
                 (println "in script, data =" data)
                 [(ops/assign :count (inc (:count data)))])})
```

---

### 10. The variable name `cart-chart` is a leftover from an earlier exercise version

**What you'll see.** The exercise stub and solution both name the chart `cart-chart`, but the topic is a counter. The test references `cart-chart`. You wonder whether you should rename.

**What's really happening.** The exercise was likely cloned from a different scenario where the name made sense. The current docstring focuses on counter operations; the variable name didn't get updated. Strictly cosmetic.

**Defense.** Don't rename. The test references this specific variable name; renaming requires changing the test too, which means diverging from the upstream `statechart-exercises` repo. Treat the inconsistency as a known-and-tracked upstream wart (worth filing back to that repo as a one-line fix at some point).
