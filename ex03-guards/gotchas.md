# Gotchas — ex03: Guards (Conditions) & Document Order

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex03. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01 and ex02 gotchas still apply. This file adds what guards + the data model + `goto-configuration!` newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25 except where flagged otherwise.

---

### 1. `goto-configuration!` includes `:ROOT` in the configuration set — `start!` doesn't

**What you'll see.** ex01 and ex02 established that `(t/in? env :ROOT)` always returns `false`. Then in ex03 you write:

```clojure
(t/start! env)
(t/goto-configuration! env [(op/assign :password "secret")] #{:login})
(t/in? env :ROOT)   ; → true   (!)
```

Suddenly `:ROOT` *is* in the configuration. And it stays there even after subsequent `(t/run-events! env :go)` calls.

**What's really happening.** `goto-configuration!` builds its target configuration via `configuration-for-states` (testing.cljc), which computes "the union of `states` and their proper ancestors" — *including* the chart root. The normal `start!` lifecycle, by contrast, uses the runtime's regular entry algorithm, which excludes `:ROOT`. The two code paths produce structurally different configuration sets for the *same* logical chart-is-here state.

**Defense.** When you use `goto-configuration!`, don't write `t/in?` checks that depend on `:ROOT` being *absent*. Conversely, if a test fails because `(t/in? env :ROOT)` returned the "wrong" value, check whether the env was set up via `start!` or via `goto-configuration!`. The asymmetry is real and (as far as I can tell from the source) intentional — see [`ask_tony.md` Q3](../ask_tony.md) for the open question on whether this should be aligned.

*See also:* [tutorial Step 4](./tutorial.md), [glossary `t/goto-configuration!`](./glossary.md).

---

### 2. A guard that throws is treated as "guard returned false"

**What you'll see.** Your guard has a bug — say, `(:password (.toUpperCase data))` when `data` is `nil`, or a `(/ 0 (:count data))`. The guard throws. You expect either an exception bubbling out of `(t/run-events! …)` or the chart to halt. Instead, the chart moves on as if the guard had returned false — falling through to the next transition.

```clojure
;; Guard throws
(transition {:event :submit
             :cond  (fn [_ _] (throw (ex-info "boom" {})))
             :target :authenticated})
;; Unguarded fallback
(transition {:event :submit :target :rejected})

;; (t/run-events! env :submit) returns normally
;; (t/in? env :rejected) => true
;; The exception is logged at DEBUG level but not propagated.
```

**What's really happening.** The runtime wraps guard evaluation in something that turns exceptions into a "this guard didn't pass" signal. The DEBUG log line records the exception; the chart continues with the next matching transition.

This is a *significant* footgun: a buggy guard looks like a working chart with surprising behavior. Tests pass for the wrong reason.

**Defense.** Two practices:

1. **Keep guards pure and defensive.** Don't call methods on values that might be `nil`. `(= (:password data) "secret")` is safe; `(.toUpperCase (:password data))` is not. The `=` form handles `nil` gracefully; the method-call form throws.
2. **When debugging a chart that ends up in the "wrong" branch despite a passing guard,** check the DEBUG output for stack traces. Or temporarily wrap the guard body in `try`/`catch` and `println` the exception:

```clojure
(fn [env data]
  (try
    <your guard logic>
    (catch Throwable t
      (println "GUARD THREW:" (.getMessage t))
      false)))
```

*See also:* [`ask_tony.md` Q7](../ask_tony.md) — open question on whether this catch-and-treat-as-false behavior is contractual or incidental.

---

### 3. `data` includes `:_event`, *not* `env`

**What you'll see.** You want to read the event payload inside a guard. You reach for `env` (it's the "environment," surely the event is in there). You write `(get-in env [:event :data :password])` and get `nil`. Your guard returns the wrong thing; the chart misbehaves.

**What's really happening.** The current event lives at `(get data :_event)`, not in `env`. The `env` map contains *runtime infrastructure* (the working-memory store, the data model implementation, the event queue, etc.) — none of which is the event itself.

The mnemonic that helps: **`env` is *plumbing*, `data` is *content*.** Anything you'd reason about as "what's the chart looking at right now" — session data, event payload, event name — is in `data`. Anything you'd reason about as "what tool would I use to manipulate the chart" — the data model, the event queue, the processor — is in `env`.

**Defense.** Always reach into `data` first, never `env`, when you want event-time information. The event name is at `(:name (:_event data))` (or equivalently `(get-in data [:_event :name])`); the event payload is at `(:data (:_event data))`.

*See also:* [tutorial Step 1](./tutorial.md), [glossary `:_event`](./glossary.md).

---

### 4. Session data starts as `nil`, and `nil` propagates through key access

**What you'll see.** You write `(fn [_ data] (= (:password data) "secret"))`. You forget to set up the data model (no `goto-configuration!`, no `on-entry` `op/assign`). The guard runs; `(:password nil)` returns `nil`; `(= nil "secret")` returns `false`. Guard fails; chart falls through. No error.

**What's really happening.** Before any operation writes to the data model, the value at `::sc.dm/data-model` is literally `nil`. The runtime still passes `data` to the guard, but `data` will essentially be `{:_event {...}}` — no session-data keys at all.

**Defense.** Before driving a chart through a scenario that depends on session data, *verify* the data is there. In tests, that's either `goto-configuration!` with an `op/assign` op, or driving the chart through whatever production sequence (e.g., events with payload) seeds the data. At the REPL, you can introspect:

```clojure
(-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
    :storage deref :test
    (get :com.fulcrologic.statecharts.data-model.working-memory-data-model/data-model))
```

If that's `nil`, your guard can never see session data.

---

### 5. A missing `:cond` is "unconditionally true," not "missing"

**What you'll see.** You write a transition without a `:cond`:

```clojure
(transition {:event :submit :target :rejected})
```

Then you wonder whether it always fires or never fires. You're considering whether to add `:cond (fn [_ _] true)` to "make sure" it always fires.

**What's really happening.** A missing `:cond` means the transition is **unconditionally enabled**. You do *not* need `(fn [_ _] true)`; the absent `:cond` is treated the same way. This is what makes the [fallback pattern](./tutorial.md) terse.

The opposite mistake — adding `:cond (fn [_ _] false)` — does what you'd expect: the transition is permanently disabled, and the runtime will always skip it.

**Defense.** When reading someone else's chart, treat a transition with no `:cond` as "always enabled on this event." Don't add a tautology guard; it adds noise without adding meaning.

---

### 6. Guards on inactive states are *not* called

**What you'll see.** You're debugging a chart with several states. You put a `println` inside a guard on state `:other` to confirm it runs. You fire `:submit` while the chart is in `:login`. Your `println` never fires.

**What's really happening.** The runtime only evaluates guards on transitions belonging to *currently active* states (per ex02's ancestor walk). If the chart isn't in `:other` or one of its descendants, transitions declared on `:other` are simply not in the candidate set — their `:cond` functions never run.

**Defense.** When a guard isn't running, the first question is "is the chart actually in the state where this guard's transition lives?" Use `(t/in? env <state-id>)` to confirm. If you're chasing a bug from logs and your guard never seems to log, the bug isn't *in* the guard — it's that the runtime hasn't reached your guard's transition.

*This is a feature, not a bug.* It's why charts scale: thousands of transitions across a complex chart don't all run on every event — only the handful belonging to currently-active states do.

---

### 7. Document order matters now in a way it didn't in ex01/ex02

**What you'll see.** ex01 had one transition per event per state. ex02 had multiple events but distinct names per transition. ex03 introduces *multiple transitions on the same event in the same state*. Reordering them changes which one fires.

**What's really happening.** The runtime always evaluates transitions within a state in document order; this rule was there in ex01 and ex02, but it didn't *matter* because no two transitions in those exercises competed for the same event. ex03 is the first exercise where the rule has practical consequences.

**Defense.** When refactoring a chart, never alphabetize or "tidy up" transition order without verifying tests still pass. The order is semantic. If you must reorder for readability, run the test suite immediately. Comments like `;; ORDER MATTERS — guarded first, unguarded fallback` on the source-side aren't a bad habit either.

---

### 8. Guard arity-1 vs arity-2 confusion

**What you'll see.** You see Clojure code conventions like `(fn [data] (= (:password data) "secret"))` in the wild and try to use that in a `:cond`. The chart errors with `Wrong number of args (2) passed to: <your-fn>`.

**What's really happening.** The runtime *always* calls the guard with two args: `env` and `data`. A 1-arg function will error when called with 2 args.

**Defense.** Always declare guards with the 2-arg signature. If you don't need `env`, name it `_env` (or just `_`):

```clojure
(fn [_env data] (= (:password data) "secret"))
;; or
(fn [_ data] (= (:password data) "secret"))
```

This is the convention the solution file uses.

---

### 9. The exercise file *already* requires `op` — but not `state`/`transition`

**What you'll see.** You skim the `(ns ...)` form, recognize `op` from the test code, and conclude all the requires are set up. Then you type `(state {:id :login} ...)` and the editor flags `state` as unresolved.

**What's really happening.** The stub's `(:require ...)` includes the *data-model.operations* namespace (because the test code uses it for `goto-configuration!` setup) but *not* the *elements* namespace. The same first-move from ex01 and ex02 applies: add `[com.fulcrologic.statecharts.elements :refer [state transition]]`.

**Defense.** Glance through every test file's `(ns ...)` against the symbols you'll use in your chart definition before you start typing. The stubs are deliberately minimal so the learner adds requires themselves; assuming they're complete bites.

---

### 10. `:authenticated` looks "final" but isn't a `final` element

**What you'll see.** Your chart enters `:authenticated`. You try to drive it further with `(t/run-events! env :next)`. The chart stays put — `:authenticated` has no outgoing transitions. You wonder whether you should use the library's `final` element instead.

**What's really happening.** `:authenticated` is just a regular atomic state that happens to have no outgoing transitions. It's "stuck" in a behavioral sense, but it's not formally a [[final-state]] — that's a distinct chart element with specific semantics (covered in ex08, including the `done.state.<id>` internal event that gets raised when one is entered).

For ex03's purposes, "no outgoing transitions" is enough. The exercise's test creates a *fresh env* (`env2`) to test the wrong-password scenario, sidestepping the stuck-ness.

**Defense.** When you genuinely want "this state terminates a region and raises a `done.state.X` event," use `(final {:id :X})`. Until then, an atomic state without transitions is the lighter-weight version.

*See also:* ex08 (final states and `done.state`).
