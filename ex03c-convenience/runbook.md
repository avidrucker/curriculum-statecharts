# Runbook — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

The operational pass. By the end you'll have edited `src/exercises/ex03c_convenience.cljc` to rewrite a login-gate chart using the convenience namespace and seen all six assertions go green.

**Estimated time:** 25–40 minutes if you've completed ex03 and ex03b. The chart is *structurally similar* to ex03's; what's new is the convenience-helper notation and the `choice` element.

---

## Prerequisites

- You've completed ex03 (guards) and ex03b (data operations).
- You've read [`tutorial.md`](./tutorial.md) for ex03c. The runbook references its terms (`on`, `choice`, eventless transition, transient state, the convenience namespace's ALPHA stability) without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` — interactive REPL | from the exercise repo root |
| **Shell** | `clojure -M:test` — the kaocha test runner | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex03c_convenience.cljc`. ~80 lines. Five things to notice:

1. **The docstring** describes the target structure as a tree and lists the three convenience helpers (`on`, `handle`, `choice`). It also calls out the contrast with ex03 — "same behavior, much less boilerplate."
2. **The `(ns …)` form** requires `clojure.test`, `statecharts.chart`, and `statecharts.testing` — but **not** the elements namespace, **not** the convenience namespace, **not** `ops`. You'll add the latter three.
3. **The `deftest`** has six assertions across two `let` blocks. The fresh-env pattern (from ex02 and ex03) reappears because the chart has one-way paths — `:authenticated` and `:rejected` don't lead back to `:idle` without an explicit `:retry`.
4. **The test fires `:enter-password` with event-payload data** — `{:name :enter-password :data {:password "secret"}}` — exercising the `(-> data :_event :data :password)` access pattern from ex03b.
5. **There's no on-entry** anywhere in the target chart. The password isn't set in session data until the user fires `:enter-password`; before that, `(:password data)` is `nil` and the choice's predicate compares `nil` to `"secret"` (which is `false`).

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex03c-convenience/convenience-test --watch
```

Summary line:

```
1 tests, 6 assertions, 6 failures.
```

(Six `is` assertions across the five `testing` blocks — one block has two.)

Keep the watch terminal open.

---

## Step 3 — Add four requires

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.convenience :refer [choice handle on]]
    [com.fulcrologic.statecharts.data-model.operations :as ops]
    [com.fulcrologic.statecharts.elements :refer [state]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Notice: we refer `state` from the elements namespace (still needed for the wrapper states) but **not** `transition`, **not** `script`, **not** `on-entry`. The convenience helpers replace those in this chart.

Save. The test runner re-runs and still reports `7 failures`.

---

## Step 4 — Build the chart, skeleton first

This exercise's chart is more **interconnected** than ex01–ex03b. Every state references at least one other state by `:id`: `:idle` targets `:checking`; the choice targets `:authenticated` and `:rejected`; `:rejected` targets `:idle`. If you add transitions before their targets exist, the chart **fails to register** with `Cannot register invalid chart` at `t/new-testing-env` time.

So we build skeleton-first: **all four state declarations as empty placeholders**, then fill in their bodies one piece at a time. Four edits. The pass count rises monotonically — 3 → 4 → 5 → 6 — at each step.

### 4.1 — The skeleton (four empty state declarations)

Replace `;; YOUR CODE HERE`:

```clojure
(def login-with-convenience
  (statechart {}
    (state {:id :idle})
    (state {:id :checking})
    (state {:id :authenticated})
    (state {:id :rejected})))
```

Save.

Expected — **3 assertions pass; 3 fail**:

```
1 tests, 6 assertions, 3 failures.
```

What passes: `starts in :idle` (it's the first state, document order picks it as initial), the "in `:idle` after `:enter-password`" check (the event is dropped silently — no handler — so the chart stays in `:idle`), and "`:retry` returns to `:idle`" (env2 also never left `:idle`, since no events match anything).

What fails: the password-storage check (no handler ran, data model is `nil`), and both "moved to target" checks for env1 (`:authenticated`) and env2 (`:rejected`).

### 4.2 — Add `:idle`'s handler (just the handler, no transitions yet)

```clojure
(state {:id :idle}
  (handle :enter-password
    (fn [_ data]
      (let [pw (-> data :_event :data :password)]
        [(ops/assign :password pw)])))
  )
```

(Replace the empty `(state {:id :idle})` declaration. Keep the other three states empty; **do not add `(on :submit :checking)` yet** — that comes in 4.3.)

Save.

Expected — **4 assertions pass; 2 fail**:

```
1 tests, 6 assertions, 2 failures.
```

The new pass: the password-storage check. `handle`'s targetless transition runs the script when `:enter-password` arrives, the script writes `:password "secret"` (or `"wrong"` in env2's case) to the data model. The chart stays in `:idle` because `handle` doesn't move.

The two remaining failures: both "moved to target" checks. The chart can't yet reach `:authenticated` or `:rejected` — there's no `:submit` transition out of `:idle`.

### 4.3 — Add `:idle`'s submit transition *and* `:checking`'s choice (together)

```clojure
(state {:id :idle}
  (handle :enter-password
    (fn [_ data]
      (let [pw (-> data :_event :data :password)]
        [(ops/assign :password pw)])))
  (on :submit :checking))                         ; ← new

(state {:id :checking}                            ; ← replacing the empty placeholder
  (choice {:id :auth-check}
    (fn [_ data] (= (:password data) "secret")) :authenticated
    :else :rejected))
```

These two have to be added in the same edit. Why? If you add `(on :submit :checking)` first but leave `:checking` empty, env2 still leaves `:idle` on `:submit` — but lands in an empty `:checking` and stays. That would *regress* the `:retry returns to :idle` test (which currently passes only because env2 never moves). Adding both at once gets you a chart where every transition has somewhere meaningful to go.

Save.

Expected — **5 assertions pass; 1 fails**:

```
1 tests, 6 assertions, 1 failures.
```

Two new passes: both "moved to target" checks. The choice dispatches correctly — `password = "secret"` lands at `:authenticated`; `:else` lands at `:rejected`.

The remaining failure: `:retry` from `:rejected` doesn't return to `:idle`. `:rejected` is still an empty placeholder, so it accepts no events.

### 4.4 — Fill in `:rejected` (add the retry transition)

```clojure
(state {:id :rejected}
  (on :retry :idle))
```

Save.

Expected — **all 6 assertions pass**:

```
1 tests, 6 assertions, 0 failures.
```

What just happened, end-to-end (for the success path):

1. **`:idle` starts active.** First state in document order.
2. **`(t/run-events! env {:name :enter-password :data {:password "secret"}})`** fires. `:idle`'s `handle :enter-password` matches. Its expr reads `:password` from the event payload, returns `[(ops/assign :password "secret")]`. The runtime writes the password. Chart stays in `:idle`.
3. **`(t/run-events! env :submit)`** fires. `:idle`'s `(on :submit :checking)` matches. The chart exits `:idle`, enters `:checking`. Entering `:checking` triggers entering its initial child — the `:auth-check` choice (the only child, by document order).
4. **`:auth-check`'s eventless transitions evaluate immediately.** The first one's `:cond` is `(= (:password data) "secret")` — true. The transition fires; chart exits `:auth-check`, exits `:checking`, enters `:authenticated`.
5. **`(t/in? env :authenticated)` returns `true`** ✓.

You've solved ex03c.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect (`clojure -M:nrepl` then your editor, or plain `clojure`):

```clojure
(require '[exercises.ex03c-convenience :as ex03c] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
(require '[com.fulcrologic.statecharts.convenience :refer [on choice handle]])
(require '[com.fulcrologic.statecharts.elements :refer [transition]])
```

### Probe 1: `on` vs raw `transition`

**Scenario.** You want to verify `on` is really pure sugar.

```clojure
(on :submit :checking)
;; => {:id :transition... :event :submit :target [:checking] :type :external :node-type :transition}

(transition {:event :submit :target :checking})
;; => {:id :transition... :event :submit :target [:checking] :type :external :node-type :transition}
```

Expected: structurally identical maps (different `:id`s due to auto-generation, but otherwise same shape). The runtime can't tell them apart.

**Why this probe exists.** When you read someone else's chart that mixes `on` and `transition`, knowing they're equivalent is what makes the mental model consistent.

### Probe 2: `choice` is a state

```clojure
(choice {:id :test-choice}
  (fn [_ _] true) :a
  :else :b)
```

Expected:

```clojure
{:id :test-choice
 :diagram/prototype :choice
 :node-type :state                  ; ← a STATE
 :children [{:cond ... :target [:a] :node-type :transition}
            {:target [:b] :node-type :transition}]}
```

Notice `:node-type :state` and `:diagram/prototype :choice`. The runtime treats this as a state element; the `:diagram/prototype` is a hint for chart-rendering tools.

**Why this probe exists.** "Choice is a special chart construct" is a misconception. Knowing it's a state-shaped value lets you reason about it like any other state — it has an `:id` you can query, an active-or-not status, etc.

### Probe 3: `:_event` in choice predicates

**Scenario.** You wonder whether a choice's predicate can read the event that triggered the transition into the choice's parent state.

```clojure
(def captured (atom nil))

(def chart-spy
  (com.fulcrologic.statecharts.chart/statechart {}
    (com.fulcrologic.statecharts.elements/state {:id :start}
      (on :go :decide))
    (com.fulcrologic.statecharts.elements/state {:id :decide}
      (choice {:id :inner-choice}
        (fn [_ data] (reset! captured (:_event data)) true)
        :end))
    (com.fulcrologic.statecharts.elements/state {:id :end})))

(def env (t/new-testing-env {:statechart chart-spy
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env {:name :go :data {:carries "this"}})
@captured
```

Expected:

```clojure
{:type :external
 :name :go
 :data {:carries "this"}
 :com.fulcrologic.statecharts/event-name :go}
```

The `:_event` from the triggering `:go` carries into the choice's predicate. So you *can* dispatch on event payload inside a choice — but only when the choice was reached via an event. (On initial chart entry, with no triggering event, `:_event` is `nil` — see [gotchas.md #4](./gotchas.md).)

**Why this probe exists.** A common real-world pattern is "dispatch on event payload" — e.g., a `:command` event with `{:action :create}` vs `{:action :delete}`. Knowing the choice's predicates see the triggering event lets you collapse multiple `handle`s into a single `choice` for clean dispatch.

### Probe 4: `:checking` and `:auth-check` are transient

**Scenario.** You want to confirm the chart never observably "stops" in `:checking` or `:auth-check`.

```clojure
(def env (t/new-testing-env {:statechart ex03c/login-with-convenience
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env {:name :enter-password :data {:password "secret"}})
(t/in? env :idle)              ; → true
(t/run-events! env :submit)
(t/in? env :checking)          ; → false   (passed through)
(t/in? env :auth-check)        ; → false   (passed through)
(t/in? env :authenticated)     ; → true    (where the chart landed)
```

Expected: `:checking` and `:auth-check` are both *false* after the submit cycle. The chart visited them as part of processing `:submit`, but the eventless transitions immediately moved it onward.

**Why this probe exists.** Choice states (and any state whose only outgoing transitions are eventless) are *transient*. They're real states in the chart's structure, but the chart never lingers in them. Knowing this prevents the bug of "I'll wait for the chart to be in `:checking` before firing the next event."

---

## Try breaking it

### Break 1: Omit `:else` from the choice — and let the password be wrong

**Edit.** Drop the `:else :rejected` clause:

```clojure
(state {:id :checking}
  (choice {:id :auth-check}
    (fn [_ data] (= (:password data) "secret")) :authenticated))
;; no :else
```

**Predicted symptom (for the wrong-password test, env2).** When the password is `"wrong"`, the choice's only predicate returns `false`. With no `:else`, the choice has no eligible eventless transition. **The chart gets stuck in `:auth-check`.** The test for `→ :rejected` fails; subsequent events (like `:retry`) don't apply because the chart isn't in `:rejected`. Worse, `(t/in? env :auth-check)` is `true` — a real observable state.

**What this proves.** A choice without `:else` is fragile. When the predicates don't cover every case, the chart hangs in the choice state with no way out.

**Restore the `:else`** before continuing.

### Break 2: Add an `:event` to a choice's transition

**Edit.** Try to convert the choice into "event-triggered conditional dispatch":

```clojure
(state {:id :checking}
  (choice {:id :auth-check}
    (fn [_ data] (= (:password data) "secret")) :authenticated
    :else :rejected))
```

…can't actually be done from the `choice` helper itself — `choice` always emits eventless transitions. But suppose you write the verbose form:

```clojure
(state {:id :checking}
  (state {:id :auth-check}
    (transition {:event :submit :cond (fn [_ data] (= (:password data) "secret"))
                 :target :authenticated})
    (transition {:event :submit :target :rejected})))
```

**Predicted symptom.** The chart enters `:checking`, then enters `:auth-check` (initial child). `:auth-check`'s transitions are *event-triggered* now — they only fire when `:submit` arrives. But `:submit` was *already processed* when we transitioned into `:checking`. So the chart is stuck in `:auth-check` waiting for *another* `:submit`. The test's `(t/in? env :authenticated)` fails.

**What this proves.** Choice states need eventless transitions to dispatch *on entry*. If the transitions are event-triggered, they only fire when that event arrives — which (since you're already in the choice from processing another event) means they don't fire at all.

**Restore the choice form** before continuing.

### Break 3: Reorder the choice's clauses so `:else` comes first

**Edit.** Put `:else` before the predicate clause:

```clojure
(choice {:id :auth-check}
  :else :rejected
  (fn [_ data] (= (:password data) "secret")) :authenticated)
```

**Predicted symptom.** The chart **still passes the tests**. Looking at the source of `choice` (in `com/fulcrologic/statecharts/convenience.cljc`), the `:else` clause is filtered out of the conditional clauses and appended *last* regardless of where you put it in the call. Empirically verified: a `choice` with `:else` declared first still routes `password = "secret"` to `:authenticated` and `password = "wrong"` to `:rejected`.

**But the verbose form has no such forgiveness.** If you wrote the expanded chart by hand:

```clojure
(state {:id :auth-check :diagram/prototype :choice}
  (transition {:target :rejected})                                    ; :else first in document order
  (transition {:cond (fn [_ data] (= (:password data) "secret"))
               :target :authenticated}))
```

…the unconditional transition would fire first regardless of password (document-order rule from ex02 — first matching transition wins; unconditional matches everything). The chart would always go to `:rejected`.

**What this proves.** Sugar can hide ordering subtleties. The convenience helper does internal clause-reordering for you so you can write `:else` anywhere; the verbose form does *not*. If you read someone else's chart that uses the verbose form, document order is the rule. If you mix the two carelessly, the relative ordering depends on which form you used.

**Restore the standard order** (`:else` last) before continuing — even though `choice` is forgiving, the convention is what readers expect.

---

## Common breakages

### `Cannot register invalid chart` when you build the chart

A target keyword doesn't exist as a state's `:id`. With convenience helpers, this most often happens when you wrote `(on :retry :idle)` before adding `:idle` to the chart, or you have a typo (`:retrt` instead of `:retry`). Grep for `:target` references (`grep -E ':target|on :[a-z]+ :|choice' …`) and confirm each target exists.

### `Wrong number of args (1) passed to: handle`

You wrote `(handle (fn [_ data] ...))` without the event keyword. `handle`'s signature is `(handle event expr)` — two args.

### The chart hangs after `:submit` — `(t/in? env :auth-check)` is `true`

The choice has no `:else` and the predicates returned all-falsy. See Break 1 above for the empirical demonstration. Fix: add `:else <some-state>`.

### `:enter-password` doesn't store the password

The `handle`'s expr returns a single op instead of a vector. ex03b gotcha #1 reappears here — `(handle :enter-password (fn [_ data] (ops/assign :password (-> data :_event :data :password))))` (no brackets) silently does nothing. Always: `[(ops/assign ...)]`.

### The choice's predicate sees `:password` as `nil`

The test fired `:submit` without first firing `:enter-password`, or fired them out of order. Confirm the test setup runs `:enter-password` (with payload) *before* `:submit`. The choice has nothing in `(:password data)` until something puts it there.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex03c_convenience.cljc` requires `statecharts.elements` (state), `statecharts.convenience` (on, handle, choice), `statecharts.data-model.operations` (ops).
- [ ] The chart has four top-level states (`:idle`, `:checking`, `:authenticated`, `:rejected`).
- [ ] `:checking` has one child — the `(choice {:id :auth-check} ...)` element.
- [ ] The choice has an `:else` clause.
- [ ] `clojure -M:test --focus exercises.ex03c-convenience/convenience-test` reports `6 assertions, 0 failures.`.
- [ ] You can mentally rewrite each `handle`, `on`, and `choice` in the verbose form (`transition + script` / `transition with target` / `state with eventless transitions`).
- [ ] You can predict what happens if you omit `:else` and the password is `"wrong"` (chart hangs in `:auth-check`).

**Explain in one breath:**

> *"`handle :enter-password` stores the event's payload password in session data. `on :submit :checking` moves to `:checking`. Entering `:checking` immediately enters the `:auth-check` choice; its eventless predicates evaluate; if `:password = "secret"` the chart goes to `:authenticated`, else (via `:else`) to `:rejected`. `on :retry :idle` from `:rejected` resets the flow."*

If that's natural, ex03c is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex04 (parallel — the first dense module).
