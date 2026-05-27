# Runbook — ex09: Child Chart Invocations

The operational pass. By the end you'll have edited `src/exercises/ex09_invocations.cljc` to define a parent checkout chart that invokes a payment child chart and seen all four assertions go green.

**Estimated time:** 45–75 minutes if you've completed ex01–ex08. The chart is medium-size but the testing approach is entirely different from previous exercises — manual orchestration via protocols rather than `t/run-events!` / `t/in?`. Expect to spend most of the time reading and understanding the test rather than writing chart code.

---

## Prerequisites

- You've completed ex01–ex08. Comfortable with parallel states, final states, `done.state.<id>` auto-events (ex08), and the chart-as-data model.
- You've read [`tutorial.md`](./tutorial.md) for ex09. The runbook references `invoke`, `:invokeid`, child sessions, `done.invoke.<id>`, `simple/simple-env`, and the protocol-level orchestration without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex09_invocations.cljc`. ~110 lines (longer than previous exercises because the test does manual orchestration).

Four things to notice:

1. **The child chart `payment-chart` is provided** at the top of the file. Don't modify it. It's a 2-state chart: `:charging` → (final) `:charged`.
2. **The TODO comment** spells out the structure: parent `:checkout` with `:collecting-info`, `:processing-payment` (with invoke), `:confirmation`, and a final `:done`. The invoke spec: `:id :payment-process :type :statechart :src :payment-chart`. The done-event transition: `:done.invoke.payment-process → :confirmation`.
3. **The `(ns …)` form** has many requires:
   - `com.fulcrologic.statecharts` aliased as `sc` (for the `::sc/keys` like `::sc/configuration`, `::sc/processor`)
   - `statecharts.events` (for `new-event`)
   - `statecharts.protocols` (for `sp/start!`, `sp/process-event!`, etc.)
   - `statecharts.simple` (for `simple/simple-env` and `simple/register!`)
   - **No `t/...` — no testing helpers used!**
   - You'll need to add `statecharts.elements` and refer `final`, `invoke`, `state`, `transition`.
4. **The `deftest`** is **dramatically different** from earlier exercises:
   - Uses `simple/simple-env` instead of `t/new-testing-env`.
   - Registers BOTH parent and child charts via `simple/register!`.
   - Starts the parent manually via `sp/start!`, saves its working memory.
   - Asserts using `(contains? (::sc/configuration wmem) :state-id)` instead of `(t/in? env :state-id)`.
   - Processes events with `(sp/process-event! processor env wmem (new-event :event-name))` — saves the result manually.
   - Polls the event queue with `sp/receive-events!` to handle the auto-routed `done.invoke` event.

The test's comments explain *why* (MockInvocations stubs out children). Read the comments — they're load-bearing for understanding what's happening.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex09-invocations/invocation-test --watch
```

Summary line:

```
1 tests, 4 assertions, 4 failures.
```

Keep the watch terminal open.

---

## Step 3 — Add the elements require

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts :as sc]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [final invoke state transition]]
    [com.fulcrologic.statecharts.events :refer [new-event]]
    [com.fulcrologic.statecharts.protocols :as sp]
    [com.fulcrologic.statecharts.simple :as simple]))
```

The new ones: `final`, `invoke`, `state`, `transition` from `statecharts.elements`.

Save. Still `4 failures`.

---

## Step 4 — Build the chart, layer by layer

The chart is structurally a normal compound state (much like ex08's pipeline) — *plus* an `invoke` inside one of its states. Three edits.

### 4.1 — Skeleton: `:checkout` with three child states (no invoke yet)

Replace `;; YOUR CODE HERE`:

```clojure
(def checkout-chart
  (statechart {}
    (state {:id :checkout}
      (state {:id :collecting-info}
        (transition {:event :pay :target :processing-payment}))
      (state {:id :processing-payment})
      (state {:id :confirmation}
        (transition {:event :finish :target :done}))
      (final {:id :done}))))
```

Save.

Expected — **2 of 4 pass**:

```
1 tests, 4 assertions, 2 failures.
```

What now passes:

- `(contains? config :collecting-info)` — parent starts at first child by document order.
- `(contains? config :processing-payment)` after `:pay` — the transition works.

What fails:

- `(some? child-wmem)` — no child chart is invoked yet, so `(get-working-memory wmstore env :payment-process)` returns nil.
- `(contains? config :confirmation)` after the done-event — no done-event transition exists, the chart stays in `:processing-payment`.

### 4.2 — Add the `invoke` element

```clojure
(state {:id :processing-payment}
  (invoke {:id :payment-process :type :statechart :src :payment-chart}))
```

Save.

Expected — **3 of 4 pass**:

```
1 tests, 4 assertions, 1 failures.
```

What's new:

- `(some? child-wmem)` — the child session now exists. When the parent enters `:processing-payment`, the invocation processor reads the chart from the registry (`:payment-chart`) and starts a child session with session-id `:payment-process` (= the invokeid).

What still fails:

- `(contains? config :confirmation)` — the test drives the child to its final, the runtime queues `done.invoke.payment-process`, but the parent has no transition handling it. The parent stays in `:processing-payment`.

### 4.3 — Add the `done.invoke.payment-process` transition

```clojure
(state {:id :processing-payment}
  (invoke {:id :payment-process :type :statechart :src :payment-chart})
  (transition {:event :done.invoke.payment-process :target :confirmation}))
```

Save.

Expected — **all 4 pass**:

```
1 tests, 4 assertions, 0 failures.
```

What just happened end-to-end:

1. Parent enters `:processing-payment`. The invoke runs: looks up `:payment-chart` in the registry, starts a child session with session-id `:payment-process`. Child's initial state is `:charging`.
2. The test fires `:charge-complete` on the *child* session (not the parent). Child transitions to `:charged` (a final). Runtime auto-fires `done.invoke.payment-process`, targeted at the parent's session.
3. The test's `sp/receive-events!` drains the event queue and routes `done.invoke.payment-process` to the parent.
4. Parent's `(transition {:event :done.invoke.payment-process :target :confirmation})` matches. Parent moves to `:confirmation`. The invoke (in the now-exited `:processing-payment`) is automatically terminated[^p5] — though the child was already in a final state, so cleanup is a no-op.

You've solved ex09. 🎉 **And you've completed the curriculum.**

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Load:

```clojure
(require '[exercises.ex09-invocations :as ex09] :reload)
(require '[com.fulcrologic.statecharts :as sc])
(require '[com.fulcrologic.statecharts.events :refer [new-event]])
(require '[com.fulcrologic.statecharts.protocols :as sp])
(require '[com.fulcrologic.statecharts.simple :as simple])
```

### Probe 1: Inspect both sessions side-by-side

```clojure
(def env (simple/simple-env))
(def processor (::sc/processor env))
(def wmstore (::sc/working-memory-store env))

(simple/register! env :checkout-chart ex09/checkout-chart)
(simple/register! env :payment-chart ex09/payment-chart)

(let [s0 (sp/start! processor env :checkout-chart {::sc/session-id :session-1})]
  (sp/save-working-memory! wmstore env :session-1 s0))

(::sc/configuration (sp/get-working-memory wmstore env :session-1))
;; => #{:checkout :collecting-info}    ← parent

(sp/get-working-memory wmstore env :payment-process)
;; => nil    ← child doesn't exist yet (invoke hasn't run)

(let [wmem (sp/get-working-memory wmstore env :session-1)
      wmem2 (sp/process-event! processor env wmem (new-event :pay))]
  (sp/save-working-memory! wmstore env :session-1 wmem2))

(::sc/configuration (sp/get-working-memory wmstore env :session-1))
;; => #{:processing-payment :checkout}

(::sc/configuration (sp/get-working-memory wmstore env :payment-process))
;; => #{:charging}    ← child is now running
```

Expected[^p2]: two sessions visible. Parent in `:processing-payment`; child in `:charging`. Both running independently.

**Why this probe exists.** Concrete visual confirmation that the child has its own session. The "two sessions, one event queue" mental model is the load-bearing intuition for invocations.

### Probe 2: Examine the `done.invoke` event when it fires

```clojure
(def captured (atom []))

;; Drive the parent to :processing-payment (which starts child)
;; Then drive the child to its final
(let [child-wmem (sp/get-working-memory wmstore env :payment-process)
      child-result (sp/process-event! processor env child-wmem (new-event :charge-complete))]
  (sp/save-working-memory! wmstore env :payment-process child-result))

(::sc/configuration (sp/get-working-memory wmstore env :payment-process))
;; => #{}    ← child has exited (entered final, fully terminated)

;; Drain the event queue
(sp/receive-events! (::sc/event-queue env) env
  (fn [_env event] (swap! captured conj event))
  {})

(first @captured)
;; =>
;; {:type :com.fulcrologic.statecharts/chart
;;  :name :done.invoke.payment-process
;;  :target :session-1
;;  :data {}
;;  :com.fulcrologic.statecharts/source-session-id :payment-process
;;  :invokeid :payment-process
;;  ...}
```

Expected[^p4]: the auto-event carries `:target` (parent session-id), `:invokeid` (which child fired it), `:source-session-id` (same as invokeid here), and `:type :com.fulcrologic.statecharts/chart`. Rich routing information.

**Why this probe exists.** Reading the `done.invoke` event's structure is the diagnostic skill when a chart "isn't completing properly." If the event is in the queue but not being routed, the bug is in the `sp/receive-events!` handler. If the event isn't in the queue at all, the child didn't reach final.

### Probe 3: Compare with `t/new-testing-env` (MockInvocations)

```clojure
(require '[com.fulcrologic.statecharts.testing :as t])

(def env-testing (t/new-testing-env
                   {:statechart ex09/checkout-chart
                    :mocking-options {:run-unmocked? true}}
                   {}))
(t/start! env-testing)
(t/in? env-testing :collecting-info)              ; => true
(t/run-events! env-testing :pay)
(t/in? env-testing :processing-payment)           ; => true
(t/in? env-testing :confirmation)                 ; => false  (child never ran)

;; The parent is stuck in :processing-payment forever.
;; The MockInvocations didn't start a real child.
```

Expected[^p3]: with `t/new-testing-env`, the parent enters `:processing-payment` but no child session exists. The auto-event never fires. The chart is stuck.

**Why this probe exists.** Knowing the failure mode of `t/new-testing-env` for invocation-bearing charts saves hours of confusion. If you're testing a real chart with invocations, `simple/simple-env` is required.

### Probe 4: Parent cancellation terminates the child

```clojure
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [final invoke state transition]])

;; A variant chart with an explicit :cancel transition
(def cancellable-checkout
  (statechart {}
    (state {:id :checkout}
      (state {:id :collecting-info}
        (transition {:event :pay :target :processing-payment}))
      (state {:id :processing-payment}
        (invoke {:id :payment-process :type :statechart :src :payment-chart})
        (transition {:event :cancel :target :collecting-info})    ; ← cancel goes back
        (transition {:event :done.invoke.payment-process :target :confirmation}))
      (state {:id :confirmation}))))

(def env2 (simple/simple-env))
(simple/register! env2 :checkout-chart cancellable-checkout)
(simple/register! env2 :payment-chart ex09/payment-chart)
(let [s0 (sp/start! (::sc/processor env2) env2 :checkout-chart {::sc/session-id :s2})]
  (sp/save-working-memory! (::sc/working-memory-store env2) env2 :s2 s0))

;; Drive parent to :processing-payment (child starts)
(let [wmem (sp/get-working-memory (::sc/working-memory-store env2) env2 :s2)
      wmem2 (sp/process-event! (::sc/processor env2) env2 wmem (new-event :pay))]
  (sp/save-working-memory! (::sc/working-memory-store env2) env2 :s2 wmem2))

(some? (sp/get-working-memory (::sc/working-memory-store env2) env2 :payment-process))
;; => true    ← child is running

;; Cancel — parent leaves :processing-payment
(let [wmem (sp/get-working-memory (::sc/working-memory-store env2) env2 :s2)
      wmem2 (sp/process-event! (::sc/processor env2) env2 wmem (new-event :cancel))]
  (sp/save-working-memory! (::sc/working-memory-store env2) env2 :s2 wmem2))

(some? (sp/get-working-memory (::sc/working-memory-store env2) env2 :payment-process))
;; => false   ← child is gone — automatically terminated when parent exited
```

Expected[^p5]: after the parent fires `:cancel` and leaves `:processing-payment`, the child session is automatically removed from the working-memory-store. The child has no awareness that it was cancelled.

**Why this probe exists.** Empirical proof of the lifecycle-coupling rule. Particularly useful when designing real charts with user-cancellable flows — knowing the child cleans itself up means you don't have to remember to call a teardown.

---

## Try breaking it

### Break 1: Drop the `:type :statechart` from invoke

**Edit.** Remove the `:type` key:

```clojure
(invoke {:id :payment-process :src :payment-chart})
```

**Predicted symptom.** Per the library source, `:type` defaults to `:statechart`. So this *should* still work. Empirical check: probably no behavioral change.

**What this proves.** `:type :statechart` is the default; omitting it is fine. Explicit type is more readable but not required.

**Restore `:type :statechart`** for clarity.

### Break 2: Wrong invokeid in the done-event transition

**Edit.** Change the transition's event:

```clojure
(transition {:event :done.invoke.payment :target :confirmation})    ; was :payment-process
```

**Predicted symptom.** The 4th assertion fails. The child still reaches its final and the runtime fires `:done.invoke.payment-process` (matching the *invoke*'s `:id`). But the transition listens for `:done.invoke.payment` — different name. The event isn't matched; the parent stays in `:processing-payment`.

(Note: ex02's string-prefix matching fallback might cause `:done.invoke.payment` to match `:done.invoke.payment-process` because the former is a string-prefix of the latter. Empirically verify — if it does, the test might "accidentally" pass via the fallback. If it doesn't, the test fails.)

**What this proves.** The auto-event name's suffix is the *invoke*'s `:id`, not anything else. Spell-check both.

**Restore the original event name** before continuing.

### Break 3: Forget to register the child chart

**Edit.** Comment out the child chart registration in the test:

```clojure
(simple/register! env :checkout-chart checkout-chart)
;; (simple/register! env :payment-chart payment-chart)        ; ← commented out
```

**Predicted symptom.** The parent reaches `:processing-payment`. The invoke runs, tries to look up `:payment-chart` in the registry, doesn't find it. Behavior depends on the invocation processor — most likely fires an `error.platform` or `error.execution` event to the parent, or silently fails to start the child. `(some? child-wmem)` returns false.

**What this proves.** `:src` is a registry key — the chart must be registered before invocation. Two charts → two registrations. (The test's setup does this correctly; the comment is for diagnosis when you encounter "child never starts" in your own code.)

**Restore the registration** before continuing.

---

## Common breakages

### `(some? child-wmem)` is false

Three likely causes:

1. **No `invoke` element in `:processing-payment`.** Confirm the chart has `(invoke {:id :payment-process :type :statechart :src :payment-chart})` inside that state.
2. **The child chart isn't registered.** Confirm both `(simple/register! env :checkout-chart ...)` and `(simple/register! env :payment-chart ...)` are called before `sp/start!`.
3. **The parent isn't in `:processing-payment`.** The invoke only runs when the enclosing state is active. Check that `:pay` was fired and the parent moved.

### `(contains? config :confirmation)` is false after the test's done-event handling

Two likely causes:

1. **The done-event transition's name is wrong.** Should be `:done.invoke.payment-process` (matching the invoke's `:id`). Typos here are silent.
2. **The `sp/receive-events!` call isn't routing the event correctly.** The test's handler function takes `{:keys [target] :as event}` and routes to the target session. If your test setup omits this, the event sits in the queue unprocessed.

### `Cannot register invalid chart` from the parent chart's construction

Either `:src` or a `:target` references a non-existent state. The most common case: the `:target` of the done-event transition (`:confirmation`) doesn't exist in your chart. Add the `:confirmation` state declaration.

### The child never gets `:charge-complete`

The test must fire `:charge-complete` on the *child* session (`:payment-process`), not the parent. From the test:

```clojure
(let [child-wmem (sp/get-working-memory wmstore env :payment-process)
      child-result (sp/process-event! processor env child-wmem (new-event :charge-complete))]
  (sp/save-working-memory! wmstore env :payment-process child-result))
```

If you accidentally call `sp/process-event!` on the parent's wmem (the parent has no `:charge-complete` handler), nothing happens.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex09_invocations.cljc` requires `statecharts.elements` and refers `final`, `invoke`, `state`, `transition`.
- [ ] `checkout-chart` has `:processing-payment` containing both an `(invoke ...)` element AND a `(transition {:event :done.invoke.payment-process :target :confirmation})`.
- [ ] `clojure -M:test --focus exercises.ex09-invocations/invocation-test` reports `4 assertions, 0 failures.`.
- [ ] You can articulate why the test uses `simple/simple-env` instead of `t/new-testing-env`.
- [ ] You ran Probe 4 (parent cancellation) and saw the child get auto-terminated.
- [ ] You can rewrite the test's manual `sp/process-event!` + `sp/save-working-memory!` sequence in your head as "what `t/run-events!` would do for a non-invocation chart."

**Explain in one breath:**

> *"When the parent enters `:processing-payment`, the `invoke` element starts the `:payment-chart` as a child session with session-id `:payment-process`. The child runs independently until its `:charge-complete` event drives it to a final. The runtime then auto-fires `:done.invoke.payment-process` (the invoke's `:id` is the suffix), routed to the parent's session. The parent's transition catches it and moves to `:confirmation`. If the parent ever leaves `:processing-payment` without the done-event, the child is auto-terminated."*

If that's natural, you've completed the curriculum. Move on to [`quiz.md`](./quiz.md) and [`gotchas.md`](./gotchas.md) for the wrap-up checks.

---

> **Verified empirically (probes run during ex09 authoring):**
>
> - [^p2]: After parent enters `:processing-payment`, child session exists with its own config `#{:charging}`
> - [^p3]: `t/new-testing-env` doesn't start child charts (MockInvocations); the parent gets stuck
> - [^p4]: `done.invoke.<invokeid>` event has `:target` (parent session-id), `:invokeid`, `:source-session-id`, `:type :com.fulcrologic.statecharts/chart`
> - [^p5]: Parent exiting the invoking state auto-terminates the child session (working memory removed)
