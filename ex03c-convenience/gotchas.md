# Gotchas — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex03c. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex03b gotchas still apply. This file adds what `choice`, `on`, eventless transitions, and the deeper convenience-namespace surface newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. A `choice` without `:else` whose predicates all fail leaves the chart stuck in the choice state

**What you'll see.** You write a `choice` covering "the obvious cases" but forget `:else`. A test scenario produces data the predicates don't cover. The chart enters the choice and *never moves*. `(t/in? env :authenticated)` is false; `(t/in? env :rejected)` is false; `(t/in? env :auth-check)` (the choice's ID) is **true**.

```clojure
(choice {:id :decide}
  (fn [_ data] (= (:x data) 1)) :a
  (fn [_ data] (= (:x data) 2)) :b)
;; no :else!
;; If (:x data) is 3, the chart enters :decide and never leaves.
```

**What's really happening.** A `choice` emits a state whose only outgoing transitions are eventless and gated by `:cond`. If no `:cond` returns truthy and there's no unconditional fallback, no transition is eligible. The chart sits in the choice. Subsequent events that don't match a transition somewhere else in the active configuration are silently dropped (per ex01 / ex02 gotchas).

**Defense.** **Always include `:else <some-state>` in a `choice`** unless you can prove every reachable data-model shape is covered by the conditional clauses. The cost of an unnecessary `:else` is one extra line; the cost of a missing one is a silently-hanging chart.

A defensive idiom for "I really do cover everything":

```clojure
(choice {:id :decide}
  (fn [_ data] (= (:x data) 1)) :a
  (fn [_ data] (= (:x data) 2)) :b
  :else :unexpected-state-id)        ; takes you somewhere with a log / error / retry
```

…and then `:unexpected-state-id` does whatever your application wants for "I got unexpected data."

*See also:* [tutorial Step 7](./tutorial.md), [runbook "Try breaking it" #1](./runbook.md), [glossary `stuck-in-choice failure mode`](./glossary.md).

---

### 2. `choice`'s `:id` is a real state ID — collisions and shadowing apply

**What you'll see.** You declare `(choice {:id :check} ...)` and elsewhere `(state {:id :check} ...)`. The chart fails to load with `Duplicate element ID on chart: :check`, or worse, one quietly overrides the other depending on element ordering.

**What's really happening.** `choice` emits a `state` element with the `:id` you gave it. State IDs must be unique within the chart (the `statechart` factory checks this and throws `Duplicate element ID on chart: <id>` if not). The `:diagram/prototype :choice` marker doesn't carve out a separate ID namespace.

**Defense.** Treat a choice's `:id` like any other state's `:id`. Pick names that signal "this is a choice" without colliding with normal states. A common convention: `:check-X`, `:decide-X`, `:dispatch-X`, `:branch-X` for choices; bare nouns (`:idle`, `:checking`, `:authenticated`) for normal states.

---

### 3. `(t/in? env :choice-id)` is almost always false — and that's the success case

**What you'll see.** You write `(is (t/in? env :auth-check))` thinking "the chart should land in the choice." The assertion fails. The chart actually went *through* the choice to the target.

**What's really happening.** Choice states are transient. They exist only during the event-processing cycle that enters them; the eventless transitions fire immediately, the chart exits, and post-settling the choice is no longer in the configuration. Tests that assert a state-after-processing should target the choice's *target* (the post-dispatch state), not the choice itself.

**Defense.** Assert on the final state (the target of the choice's transitions), not on the choice itself. If you genuinely need to know the chart is "passing through" a choice — typically for diagnostic logging — add an `on-entry` to the choice that logs:

```clojure
(state {:id :checking}
  (state {:id :auth-check :diagram/prototype :choice}
    (on-entry {} (script {:expr (fn [_ data] (println "checking with" (:password data)) [])}))
    (transition {:cond ... :target :authenticated})
    (transition {:target :rejected})))
```

…which is no longer the convenience `choice` form but the verbose equivalent. The flexibility is the point.

---

### 4. `:_event` is `nil` inside a choice predicate that the chart entered automatically

**What you'll see.** You have a choice as the chart's initial state (or as the initial child of a compound state entered without an external event). Your predicate does `(-> data :_event :data :something)` and gets nil. The predicate returns false. The chart routes to `:else` (or hangs if there's no `:else`).

```clojure
(statechart {}
  (choice {:id :initial-dispatch}
    (fn [_ data] (= :admin (-> data :_event :data :role))) :admin-home
    :else :user-home))
```

On `start!`, no event triggered the entry — this is automatic chart entry. `:_event` is `nil`. `(-> nil :data :role)` is `nil`. `(= :admin nil)` is `false`. The chart goes to `:user-home`, regardless of what was supposed to happen.

**What's really happening.** `:_event` is the runtime-injected "current event" — but with no current event (automatic entry), there's nothing to inject. The library leaves `:_event` as `nil`.

**Defense.** Don't dispatch on `:_event` in a choice that might be entered automatically. Use session data instead: `handle` the relevant event somewhere upstream to write a value into the data model, then dispatch on that value in the choice. The exercise's solution does exactly this — `handle :enter-password` writes `:password`, then the choice reads `(:password data)`.

If you *must* dispatch on event data in a choice, guard the access:

```clojure
(fn [_ data] (= :admin (some-> data :_event :data :role)))
```

`some->` short-circuits on `nil`, so a missing `:_event` returns `nil` from the threading — and `(= :admin nil)` is just false (not an NPE).

*See also:* [runbook Probe 3](./runbook.md) (auto-entry case), [tutorial Step 5 — `:_event` in eventless predicates](./tutorial.md).

---

### 5. `on`'s body must be element values, not config maps

**What you'll see.** You try to attach a script to an `on` for "transition + side effect" by passing a config map after the target:

```clojure
(on :submit :checking
  {:expr (fn [_ _] [(ops/assign :submitted-at (System/currentTimeMillis))])})
```

The chart **throws at construction** with:

```
Illegal children of :transition with attributes {...}.
That node cannot have children of type(s): (nil)
```

The error message is loud (good — not a silent footgun like Gotcha #1) but the *cause* is not obvious unless you know `on`'s shape.

**What's really happening.** `on`'s signature is `(on event target & actions)`. The variadic `actions` are *child elements* (real chart elements with `:node-type`), not raw config maps. A bare `{:expr ...}` map has no `:node-type` key, so it's "nil-typed" — and `:transition` doesn't allow nil-typed children.

The fix is to wrap the expression in a `script` element:

```clojure
(on :submit :checking
  (script {:expr (fn [_ _] [(ops/assign :submitted-at (System/currentTimeMillis))])}))
```

`(script ...)` returns a value with `:node-type :script`, which IS a legal child of a transition. The chart loads; on `:submit`, the chart transitions to `:checking` *and* runs the script.

**Defense.** When you want to attach behavior to an `on`, the body must be element-shaped (`script`, `assign`, `log`, etc.) — same as inside a verbose `transition`. The convenience helper doesn't auto-wrap a `:expr` config map into a script element.

If `on + script` becomes verbose, the verbose `transition` form reads about the same:

```clojure
(transition {:event :submit :target :checking}
  (script {:expr (fn [_ _] [(ops/assign :submitted-at (System/currentTimeMillis))])}))
```

…which is what `on` would emit anyway.

---

### 6. `choice` is a function, not a macro — but `choice-with-annotations` (in `convenience-macros`) is a macro

**What you'll see.** You read someone else's chart and see `(choice {:id ...} ...)`. You assume it's a macro because of the `cond`-like shape, try to `apply` it (`(apply choice opts clauses)`), and it... works? Then you switch to a chart using `convenience-macros/choice` and `apply` fails.

**What's really happening.** The library has *two* `choice`s:

- `com.fulcrologic.statecharts.convenience/choice` — **a function**. You can `apply` it. Dynamic clause construction works.
- `com.fulcrologic.statecharts.convenience-macros/choice` — **a macro**. Strips quoted predicates for diagram-rendering annotations. Can't be `apply`-ed.

ex03c uses the function. If you ever see `(:require [com.fulcrologic.statecharts.convenience-macros :refer [choice]])`, you've switched to the macro variant.

**Defense.** When in doubt about whether a convenience helper is a function or a macro, check the namespace it came from. The library's source files name `convenience.cljc` (functions) and `convenience-macros.cljc` (macros) consistently. The macros add chart-rendering annotations but lose `apply`-ability.

---

### 7. Comparing convenience-helper output to verbose form with `=` returns false

**What you'll see.** You want to verify `on` and `transition` are equivalent. You write:

```clojure
(= (on :submit :checking)
   (transition {:event :submit :target :checking}))
;; => false
```

And conclude they're different. They're not.

**What's really happening.** The auto-generated `:id` (`:transition17520` vs `:transition17521`) differs. The maps are otherwise identical in shape — same `:event`, same `:target`, same `:node-type`. The runtime treats them identically. But `=` compares maps element-wise, and the `:id` mismatch produces `false`.

**Defense.** Don't compare chart structures with `=`. To verify equivalence, strip the auto-IDs (`(dissoc m :id)`) or compare behaviorally (run both through `t/new-testing-env` and check the same scenarios).

*Same lesson as ex03b gotcha #7 — surfacing again here because the convenience helpers make it tempting to "verify the sugar is equivalent" via `=`.*

---

### 8. The convenience namespace's `state` element comes from elements, not convenience

**What you'll see.** You include `[com.fulcrologic.statecharts.convenience :refer [on choice handle]]` in your requires, expecting `state` to be available too. You write `(state {:id :idle} ...)` and the editor flags `state` as unresolved.

**What's really happening.** `state` is in `com.fulcrologic.statecharts.elements`, not convenience. The convenience namespace internally requires elements and uses `state`, but it doesn't re-export it. You always need a separate `(:require [com.fulcrologic.statecharts.elements :refer [state ...]])` for the wrapper states.

**Defense.** Memorize the namespace split:

- **Elements** (`com.fulcrologic.statecharts.elements`): `state`, `transition`, `parallel`, `final`, `script`, `on-entry`, `on-exit`, `history`, `invoke`, etc.
- **Convenience** (`com.fulcrologic.statecharts.convenience`): `on`, `handle`, `choice`, `assign-on`, `send-after`, etc.

ex03c's chart uses both: `state` from elements (for the wrapper containers), `on`/`handle`/`choice` from convenience (for the contents).

---

### 9. Eventless transitions can cascade — multiple states transit in one cycle

**What you'll see.** You add a chain of choice states (or other eventless-transition states) and expect the chart to "stop" at each one. Instead, the chart blasts through all of them in a single event-processing cycle, settling at the first state with no eligible eventless transition.

```clojure
(state {:id :a}
  (transition {:target :b}))                ; eventless

(state {:id :b}
  (transition {:target :c}))                ; eventless

(state {:id :c})                            ; no eventless transition — chart stops here
```

If the chart enters `:a` (e.g., from another transition), it immediately fires `:a`'s eventless transition to `:b`, then `:b`'s to `:c`, and settles in `:c`. From a test's view, `:a` and `:b` are both transient.

**What's really happening.** The runtime's "settling" loop processes eventless transitions until no more are eligible. Each settling step moves the chart one state along the eventless chain. Settling completes when the configuration is "stable" (no eligible eventless transitions remaining).

**Defense.** When chaining eventless states (e.g., `:check-auth` → `:check-permissions` → `:final-state`), be aware that the chart sees all of them as one logical move from the test's perspective. If you want to *observe* the chart in an intermediate state, that intermediate state must have at least one event-triggered transition that *doesn't fire* (no matching event was sent), so the chart parks there.

---

### 10. The chart-rendering `:diagram/prototype :choice` annotation has no runtime semantics — but tools might depend on it

**What you'll see.** You manually write the equivalent of `choice` without the convenience helper:

```clojure
(state {:id :auth-check}                   ; no :diagram/prototype!
  (transition {:cond ... :target :a})
  (transition {:target :b}))
```

Everything works. The tests pass. You assume the `:diagram/prototype` key from the `choice` helper is dead weight.

**What's really happening.** The runtime does ignore it. **But**: any chart-rendering / visualization tool (and the library's TODO'd diagram generator) uses `:diagram/prototype :choice` to know "this state should be drawn as a diamond, not a rectangle." Without the annotation, the diagram tool sees a regular state.

**Defense.** If you're authoring chart elements that conceptually behave like choices but want to skip the `choice` helper (e.g., because you need finer control over the children), add `:diagram/prototype :choice` to the state's attrs map manually:

```clojure
(state {:id :auth-check :diagram/prototype :choice}
  (transition {:cond ... :target :a})
  (transition {:target :b}))
```

It costs nothing at runtime and preserves the right rendering downstream. Trivia for now (no curriculum exercise depends on chart rendering), but worth knowing.
