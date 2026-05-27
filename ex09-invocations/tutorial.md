# Tutorial — ex09: Child Chart Invocations

The conceptual narrative behind exercise 9. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex09 is the densest module — and the last. New element (`invoke`), new auto-event (`done.invoke.<invokeid>`), new lifecycle concept (child sessions), and an entirely new testing approach (`simple/simple-env` + manual protocol calls instead of `t/new-testing-env` + `t/in?`/`t/run-events!`). Take this one slowly; the payoff is the most powerful pattern the library supports.

---

## The story we're building

A checkout flow that delegates payment processing to a **child chart**.

The parent chart (`:checkout`) collects info, then enters `:processing-payment`. Inside `:processing-payment`, the parent **invokes** a separate `:payment-chart` — a child statechart that runs its own lifecycle (`:charging` → `:charged`). When the child reaches its final state, the runtime fires `done.invoke.<invokeid>` back at the parent, which transitions to `:confirmation`.

The clean separation:
- **Parent** knows "we're doing payment now" but not *how* payment works.
- **Child** is a self-contained chart with its own states and events.
- The two communicate via the `invoke` mechanism — when, where, what to start, and how to react when it's done.

This pattern is how real Fulcro apps compose: a top-level routing chart invokes feature-specific charts that themselves invoke worker charts. The library treats child invocations as first-class.

What you'll have grounded by the end:

1. The **`invoke` element** — what it does, how `:id`, `:type`, `:src` work.
2. The **`done.invoke.<invokeid>` auto-event** — when it fires, what it contains, how to handle it.
3. **Child sessions** — each invoked chart has its own session-id (= the `invokeid`), its own working memory.
4. **Child lifecycle tied to parent state** — child is alive while the parent's invoking state is active; cancelled when the parent leaves.
5. Why this exercise can't use `t/new-testing-env` — the testing env's `MockInvocations` stubs out child charts. The exercise uses `simple/simple-env` for real execution.

---

## Step 1 — The `invoke` element spawns a child chart

The element:

```clojure
(invoke {:id :payment-process
         :type :statechart
         :src :payment-chart})
```

Returns[^p1]:

```clojure
{:id :payment-process
 :type :statechart
 :src :payment-chart
 :explicit-id? true
 :node-type :invoke}
```

Three keys to understand:

- **`:id` (the invokeid)** — a keyword that becomes the child session's session-id. When the parent's invoking state is entered, a new child session is started with this id, and all subsequent operations on the child reference this id. The same id also forms the auto-event name: `:done.invoke.<this-id>`.
- **`:type`** — the kind of invocation. `:statechart` is the only kind the curriculum uses; the library also supports `:future` (CLJ-only, for async non-chart work). Default is `:statechart`.
- **`:src`** — a registry key. The parent's environment must have a chart *registered* under this key before invocation; if not, the invoke fails silently.

`invoke` lives as a child of the state where the child should run. When the chart enters that state, the child is started. When the chart exits, the child is terminated.

The library supports many more options on `invoke` — `:params` / `:namelist` for passing data into the child, `:finalize` for processing events from the child, `:autoforward` to forward external events to the child, `:idlocation` for storing an auto-generated id. None are needed for ex09; mentioned here for awareness when you read real production charts.

> **Common misconception** — "`invoke` is like calling a function — synchronous, returns a value."
>
> It isn't synchronous and it doesn't return a value to the parent directly. It *starts* a child chart that runs independently. The parent has no idea what the child is doing; it just knows when the child reaches a final via the auto-event.

---

## Step 2 — Child has its own session-id and working memory

When the parent enters the state containing `invoke`, the runtime:

1. Looks up `:src` in the chart registry.
2. Starts the child chart with **session-id = the invokeid**.
3. Stores the child's working memory in the working-memory-store under that session-id.

From the test code:

```clojure
(simple/register! env :checkout-chart checkout-chart)    ; parent under :checkout-chart key
(simple/register! env :payment-chart payment-chart)      ; child under :payment-chart key

(sp/start! processor env :checkout-chart {::sc/session-id :checkout-session})
;; Parent now running. After parent fires :pay, the invoke runs:
;;   - Child chart looked up from registry under :payment-chart
;;   - Child started with session-id = :payment-process (the invokeid)

(sp/get-working-memory wmstore env :payment-process)
;; → child's working memory map, including its configuration
```

So **two sessions are active in parallel** at the working-memory-store level: parent under `:checkout-session`, child under `:payment-process`[^p2]. They communicate only through events, not shared state.

Empirically:

```
After parent :pay → child started:
  Parent config: #{:processing-payment :checkout}
  Child config:  #{:charging}
```

Both sessions can be inspected, advanced, and queried independently.

> **Common misconception** — "The child's states show up in the parent's configuration."
>
> They don't. The parent's configuration contains only its own states (`:checkout :processing-payment`). The child's states (`:charging`) live in a completely separate session. The two are coupled by *events*, not by shared configuration.

---

## Step 3 — Driving the child to its final fires `done.invoke.<invokeid>`

The child chart in the exercise:

```clojure
(def payment-chart
  (statechart {}
    (state {:id :charging}
      (transition {:event :charge-complete :target :charged}))
    (final {:id :charged})))
```

When `:charge-complete` fires *on the child session* (not the parent), the child moves from `:charging` to `:charged`. `:charged` is a final → the runtime auto-fires `done.invoke.<invokeid>` (just like `done.state.<id>` from ex08, but for invocations).

The event has this shape[^p4]:

```clojure
{:type :com.fulcrologic.statecharts/chart
 :name :done.invoke.payment-process
 :target :checkout-session
 :data {}
 :com.fulcrologic.statecharts/source-session-id :payment-process
 :invokeid :payment-process
 :com.fulcrologic.statecharts/event-name :done.invoke.payment-process}
```

Notable differences from `done.state.<id>` (ex08):

- **`:target`** is set to the parent's session-id. The event is *addressed* to the parent.
- **`:invokeid`** is an explicit field.
- **`:source-session-id`** records which child fired it.
- **`:type :com.fulcrologic.statecharts/chart`** — library-specific marker, not the `:internal`/`:external` from earlier events.

The parent's transition handles it the same way as any other event:

```clojure
(state {:id :processing-payment}
  (invoke {:id :payment-process :type :statechart :src :payment-chart})
  (transition {:event :done.invoke.payment-process :target :confirmation}))
```

When the auto-event reaches the parent's session, the transition fires, the parent enters `:confirmation`, the parent's invoking state exits, and the child session (already done) is cleaned up.

> **Common misconception** — "The parent automatically routes events to the child."
>
> No. The parent doesn't automatically forward anything to the child. To send an event to the child, you (or the runtime, via the auto-event mechanism) must address it to the child's session-id explicitly. The exception is `:autoforward true` on the invoke, which *does* forward external events — but the curriculum's exercise doesn't use it.

---

## Step 4 — Parent leaving the invoking state cancels the child

The invoke's lifecycle is tied to its enclosing state. **When the parent exits the state containing `invoke`, the child is automatically terminated.** Empirically verified[^p5]:

```clojure
;; Parent in :processing-payment with child running
(t/in? parent :processing-payment)              ; → true
(some? (get-child-wmem :payment-process))       ; → true

;; Parent fires :cancel (a hypothetical transition out of :processing-payment)
;; Parent moves back to :collecting-info
(t/in? parent :collecting-info)                 ; → true
(some? (get-child-wmem :payment-process))       ; → false   ← child is GONE
```

This is the chart-author's safety net: you can't "leak" child sessions. If the parent decides the user cancelled, the child is cleaned up immediately. SCXML guarantees this.

The exercise's chart doesn't have a `:cancel` transition — the only way out of `:processing-payment` is via `done.invoke.payment-process` (success). But knowing the cancel semantics is important when you build real charts where users might back out mid-flow.

> **Common misconception** — "The child runs to completion regardless of the parent."
>
> It doesn't. If the parent exits the invoking state, the child is killed mid-flight. The child has no awareness that it was cancelled; from its perspective, the event queue just stops being polled. This is intentional — the parent's lifecycle is the source of truth.

---

## Step 5 — Why this exercise can't use `t/new-testing-env`

ex01–ex08 used `(t/new-testing-env ...)` from `com.fulcrologic.statecharts.testing`. ex09 uses **`(simple/simple-env)`** from `com.fulcrologic.statecharts.simple`. The reason: `t/new-testing-env` installs `MockInvocations` — a stub that doesn't actually start child charts.

Empirically[^p3]: with `t/new-testing-env`, after firing `:pay`, the parent moves to `:processing-payment`. But no child session is created. `(get-working-memory wmstore env :payment-process)` returns `nil`. The parent waits forever for `:done.invoke.payment-process` — it never fires.

For ex09 to actually run child charts, the test uses `simple/simple-env`, which has the *real* invocation processor (`com.fulcrologic.statecharts.invocation.statechart/new-invocation-processor`).

Trade-off: `simple-env` is more realistic but requires manual orchestration. The test calls `sp/process-event!`, `sp/save-working-memory!`, and `sp/receive-events!` directly instead of the convenient `t/run-events!` / `t/in?` helpers. The runbook will walk you through this.

> **Common misconception** — "`simple/simple-env` is the upgrade — always use it."
>
> It isn't the upgrade. For exercises that *only* test parent-side behavior, `t/new-testing-env` is faster and more ergonomic. For exercises that involve real child invocations, `simple/simple-env` is necessary. Different tools, different jobs.

---

## Step 6 — Manual orchestration: the test pattern

The exercise's test doesn't use `t/run-events!` or `t/in?`. Instead it uses the runtime's protocols directly:

```clojure
;; Get a session's working memory
(sp/get-working-memory wmstore env :checkout-session)

;; Process an event for a session — returns updated wmem (does NOT save)
(sp/process-event! processor env wmem (new-event :pay))

;; Save updated wmem
(sp/save-working-memory! wmstore env :checkout-session wmem2)

;; Process events from the event queue (the queue holds messages
;; routed between sessions, like done.invoke)
(sp/receive-events! event-queue env handler-fn {})
```

The `sp/process-event!` for the parent fires `:pay`, the parent enters `:processing-payment`, the invoke runs and starts the child. The child is now alive but no events have been sent to it.

To advance the child: get *its* working memory (via session-id `:payment-process`), call `sp/process-event!` on the child's wmem, save the child's updated wmem.

To handle `done.invoke.payment-process`: it lands in the event queue. The test's `sp/receive-events!` call drains the queue and routes each event to its target session. The handler function it provides processes the event for the targeted session.

This is verbose. It's also exactly what the real runtime does — `t/run-events!` is a thin convenience wrapper over the same protocol calls. ex09's test exposes the machinery because the child-chart routing isn't covered by the convenience wrapper.

---

## Step 7 — Putting it together

The chart, in plain English:

> A checkout flow with three steps: collect info, process payment, confirm. The payment step delegates to a separate chart that runs concurrently. When the payment chart finishes, the parent gets a `done.invoke.<id>` event and advances to confirmation.

The chart, as a tree:

```
:checkout (compound)
  ├─ :collecting-info → :processing-payment on :pay
  ├─ :processing-payment
  │    ├─ invoke :payment-process (type :statechart, src :payment-chart)
  │    └─ on :done.invoke.payment-process → :confirmation
  ├─ :confirmation → :done on :finish
  └─ (final) :done
```

The chart, as Clojure (the solution):

```clojure
(def checkout-chart
  (statechart {}
    (state {:id :checkout}
      (state {:id :collecting-info}
        (transition {:event :pay :target :processing-payment}))
      (state {:id :processing-payment}
        (invoke {:id :payment-process :type :statechart :src :payment-chart})
        (transition {:event :done.invoke.payment-process :target :confirmation}))
      (state {:id :confirmation}
        (transition {:event :finish :target :done}))
      (final {:id :done}))))
```

The child chart (provided by the exercise):

```clojure
(def payment-chart
  (statechart {}
    (state {:id :charging}
      (transition {:event :charge-complete :target :charged}))
    (final {:id :charged})))
```

The runbook walks you through writing the parent chart and the test orchestration.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **What `:id` on `invoke` becomes.** The child session's session-id, AND the suffix of the auto-event (`done.invoke.<id>`).
- **What `:src` references.** A registry key. The runtime looks up the chart there. Both parent and child must be registered.
- **What the child's working memory looks like.** A separate `wmem` map, stored under the child's session-id in the working-memory-store. The child is a complete chart in its own right.
- **When the child is terminated.** When the parent exits the invoking state. There's no manual "stop the child" call — it's lifecycle-coupled.
- **Why the test uses `simple/simple-env` instead of `t/new-testing-env`.** The testing env mocks invocations; only the simple env actually runs child charts.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns specific to invocations.

---

> **Verified empirically (probes run during ex09 authoring):**
>
> - [^p1]: `(invoke {:id ... :type :statechart :src ...})` returns `{:id ... :type :statechart :src ... :explicit-id? true :node-type :invoke}`
> - [^p2]: After parent enters invoking state, child session exists with its own working memory and configuration
> - [^p3]: `t/new-testing-env` uses `MockInvocations` — child chart never actually runs; no child session created
> - [^p4]: `done.invoke.<invokeid>` event includes `:target` (parent session-id), `:invokeid`, `:source-session-id`, and `:type :com.fulcrologic.statecharts/chart`
> - [^p5]: When the parent exits the invoking state, the child session is automatically terminated (working memory removed)
