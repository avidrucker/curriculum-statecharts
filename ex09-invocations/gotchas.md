# Gotchas — ex09: Child Chart Invocations

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex09. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex08 gotchas still apply. This file adds what `invoke` + child sessions + `simple/simple-env` newly put on the table — the densest set of footguns in the curriculum.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. `t/new-testing-env` doesn't actually run child charts (MockInvocations stubs them)

**What you'll see.** You build a parent chart with an `invoke`. You test it with `t/new-testing-env` (as you did for every earlier exercise). The parent enters the invoking state. The test asserts the post-invoke state. It fails — the parent is stuck in the invoking state forever. You see no child session in the working-memory-store.

```clojure
(def env (t/new-testing-env {:statechart parent-chart ...} {}))
(t/start! env)
(t/run-events! env :pay)
(t/in? env :processing-payment)              ; → true
(t/in? env :confirmation)                    ; → false  (forever)
```

**What's really happening.** `t/new-testing-env` installs `MockInvocations` instead of the real `statechart-invocation-processor`[^p3]. The mock doesn't start child charts; it just records that an invocation was requested. Without a real child running, `done.invoke.<invokeid>` never fires.

**Defense.** For invocation-bearing charts, use **`simple/simple-env`** from `com.fulcrologic.statecharts.simple`. Manual orchestration via `sp/process-event!` / `sp/save-working-memory!` / `sp/receive-events!` replaces `t/run-events!` / `t/in?`.

Reserve `t/new-testing-env` for single-chart testing where you don't care about real invocation behavior.

*See also:* [tutorial Step 5 + 6](./tutorial.md).

---

### 2. The invokeid name leaks into the auto-event suffix — ex02's prefix matching can bite

**What you'll see.** You name the invoke's `:id` something short like `:pay`. The auto-event becomes `:done.invoke.pay`. You write a transition listening for it:

```clojure
(transition {:event :done.invoke.pay :target :confirmation})
```

Later you add a second invocation with `:id :payment` (also a parent state somewhere else). Its auto-event is `:done.invoke.payment`. **The earlier transition (`:done.invoke.pay`) matches the new event via ex02's string-prefix fallback** — `:done.invoke.pay` is a string-prefix of `:done.invoke.payment`. Your `:confirmation` transition fires when *either* invocation completes — likely not what you wanted.

**What's really happening.** This is the third time ex02's gotcha #1 bites in the curriculum (after ex06 and ex08). The string-prefix matching fallback in `name-match?` treats `:done.invoke.pay` as matching any event whose name string-prefixes that. Two invocations with related names create cross-routing bugs.

**Defense.** Choose invokeids that are full-segment-distinct: `:payment-process`, `:shipping-process`, `:refund-process` (all clean prefix-distinct). Avoid names that string-prefix each other.

*See also:* [ex02 gotchas.md #1](../ex02-event-matching/gotchas.md), [ex06 gotchas.md #7](../ex06-history/gotchas.md), [ex08 gotchas.md #2](../ex08-final-and-done/gotchas.md).

---

### 3. Forgetting to register the child chart — invoke fails with `:error.platform`

**What you'll see.** You write:

```clojure
(simple/register! env :checkout-chart checkout-chart)
;; (simple/register! env :payment-chart payment-chart)        ; ← forgot this
(sp/start! processor env :checkout-chart {...})
;; ... drive parent to invoking state
```

The parent enters the invoking state. No child session is created (`(get-working-memory wmstore env :payment-process)` returns `nil`). The chart appears to "do nothing." But: if you drain the event queue, you'll find an `:error.platform` event waiting, targeted at the parent's session.

**What's really happening.** `invoke`'s `:src` is a registry key. When the invocation processor looks it up and finds nothing, it queues an `:error.platform` event of type `:com.fulcrologic.statecharts/chart` addressed to the parent. The event sits in the queue until someone calls `sp/receive-events!` and processes it.

**Empirically verified:** with `:checkout-chart` registered but `:payment-chart` not registered, after firing `:pay`:

```
parent config: (:checkout :processing-payment)
child session exists? false
events in queue: :error.platform → :checkout-session
```

The parent didn't move forward (no `done.invoke` will ever fire — there's no child). But the error event is *observable*.

**Defense.** Two practices:

1. **Always register both parent and child before `sp/start!`:**

   ```clojure
   (let [env (simple/simple-env)]
     (simple/register! env :parent parent-chart)
     (simple/register! env :child-A child-A)
     (simple/register! env :child-B child-B)
     ...)
   ```

2. **Add an error-handling transition for `:error.platform`** on states with invocations:

   ```clojure
   (state {:id :processing-payment}
     (invoke {:id :payment-process :type :statechart :src :payment-chart})
     (transition {:event :done.invoke.payment-process :target :confirmation})
     (transition {:event :error.platform :target :error-state}))   ; ← catch startup failures
   ```

   This catches the "child failed to start" case and routes the chart to error recovery instead of leaving it stuck.

When debugging "child never starts," (a) check that the chart is registered AND `:src` matches the registration key, then (b) drain the event queue and look for `:error.platform`.

---

### 4. The auto-event's `:target` is the *parent's* session-id, not the invokeid

**What you'll see.** You're writing custom event routing logic (e.g., a logger that captures done-events). You expect `(:target event)` on a `done.invoke.payment-process` to be `:payment-process`. It's not — it's the parent's session-id (e.g., `:checkout-session`).

**What's really happening.** The auto-event's `:target` is the **addressee** of the event — i.e., which session should process it. For `done.invoke`, that's the parent. The child's id is in `:invokeid` and `::sc/source-session-id` instead.

**Defense.** Read the event keys in order:

- `:target` — who should process this event (almost always the parent for `done.invoke`).
- `:invokeid` / `::sc/source-session-id` — who triggered the event (the child).
- `:name` — the event name (with the invokeid as suffix).

If your logger does `(println (:target event))`, you'll see parent session-ids for done-events. That's correct.

---

### 5. Child sessions are siblings, not children, in the working-memory-store

**What you'll see.** You expect the child's working memory to be nested inside the parent's. You look at the storage:

```clojure
(deref (:storage (::sc/working-memory-store env)))
;; => {:checkout-session {...parent...}
;;     :payment-process  {...child...}}
```

Two top-level entries, not nested.

**What's really happening.** The working-memory-store is a flat map from session-id to working memory[^p2]. Parent and child sessions are stored side-by-side; there's no parent-child hierarchy in the store. The relationship lives in the chart's `invoke` element, not the storage layout.

**Defense.** When iterating the working-memory-store (rare but happens in custom tooling), don't assume hierarchy. The session-ids are the only structure.

---

### 6. Parent exits the invoking state → child terminates, but child has no awareness

**What you'll see.** You design a child chart that does cleanup in its on-exit. The parent cancels the flow. You expect the child's on-exit to run. It might or might not — and there's no way to coordinate.

**What's really happening.** When the parent exits the invoking state, the runtime removes the child's working memory from the store. The child's on-exit *may* run as part of the standard exit sequence, depending on the chart implementation. But the child can't fire follow-up events — its session is gone.

Empirically[^p5]: after the parent fires `:cancel` from a probe chart, `(get-working-memory wmstore env :payment-process)` returns nil. The child is effectively erased.

**Defense.** Don't rely on the child to coordinate cleanup via events. If you need to know "the parent cancelled," let the parent fire an explicit cancellation event *before* leaving the invoking state — that gives the child one event-processing cycle to do whatever cleanup it can. But generally: keep child charts self-contained and idempotent; don't pin teardown responsibilities to them.

---

### 7. The done-event's invokeid suffix must exactly match the invoke's `:id`

**What you'll see.** Your invoke is `{:id :payment-process ...}`. You write the transition `(transition {:event :done.invoke.payment :target ...})` (missing `-process`). The child completes; the runtime fires `:done.invoke.payment-process`; your transition listens for `:done.invoke.payment`.

In a *strict* implementation, these are different keywords; no match; the parent stays stuck.

In *this library*, ex02's string-prefix matching fallback applies: `:done.invoke.payment` IS a string-prefix of `:done.invoke.payment-process`, so they DO match. Your transition fires. **But that's by accident, not by design.**

**Defense.** Spell the suffix exactly. The invoke's `:id` and the transition's `:event` suffix must match character-for-character. Don't rely on the string-prefix fallback to "save" you — it'll get you in trouble when you add another invocation later (see gotcha #2).

---

### 8. `simple/simple-env` requires manual event-queue draining

**What you'll see.** You set up `simple/simple-env`, register charts, start the parent, fire events on the parent and child. The child reaches its final. The parent doesn't move. You're confused.

**What's really happening.** Unlike `t/run-events!` (which auto-drains the queue), `simple/simple-env`'s manually-polled queue requires explicit `sp/receive-events!` calls to process queued events. The `done.invoke` event is queued by the child's final-entry but stays there until you drain it.

**Defense.** After any sequence of operations that could enqueue inter-session events, call:

```clojure
(sp/receive-events! (::sc/event-queue env) env
  (fn [{::sc/keys [working-memory-store processor] :as env}
       {:keys [target] :as event}]
    (when-let [wmem (sp/get-working-memory working-memory-store env target)]
      (let [result (sp/process-event! processor env wmem event)]
        (sp/save-working-memory! working-memory-store env target result))))
  {})
```

This drains the queue and routes each event to its `:target` session. The exercise's test does this exactly; your own tests should too.

---

### 9. Don't pass chart values for `:src`; pass keys

**What you'll see.** You write `(invoke {:id :pp :type :statechart :src payment-chart})` — passing the chart value directly (the result of `(statechart {} ...)`). The chart might construct without error, but the invocation fails when it tries to look up the chart in the registry.

**What's really happening.** `:src` is a registry *key*, not a chart value. The runtime does `(simple/lookup-statechart env src)` (or equivalent) at invocation time. Passing the chart map directly causes the lookup to fail.

**Defense.** Always: `:src :keyword-name`, where `:keyword-name` matches a `(simple/register! env :keyword-name chart-value)` call. The two-step (register, then reference by key) is the SCXML-aligned indirection — production charts often need to swap implementations behind the same key.

---

### 10. `:autoforward true` is rarely what you want

**What you'll see.** You add `:autoforward true` to the invoke thinking it'll save event-routing boilerplate. Suddenly every event the parent sends — including unrelated UI events — also goes to the child. The child handles events it shouldn't see; your child's behavior gets weird.

**What's really happening.** `:autoforward` is a coarse "forward everything external to the child" flag. It's useful for charts where the child is logically a continuation of the parent's event vocabulary (rare). It's *not* useful for charts where parent and child have distinct event vocabularies (the common case).

**Defense.** Leave `:autoforward` off (the default) unless you have a clear reason. If the parent and child need to exchange events, do it explicitly via `send` / explicit routing — clearer intent, fewer surprises.

The exercise doesn't use `:autoforward`. The parent fires `:pay`; only the parent's transition handles it. The child fires `:charge-complete` (in the test); only the child's transition handles it. Cross-routing only happens via the runtime's `done.invoke` auto-event.

---

> **Verified empirically (probes run during ex09 authoring + review pass):**
>
> - [^p2]: Working-memory-store has parent and child as sibling entries, not nested
> - [^p3]: `t/new-testing-env` doesn't start child charts (MockInvocations stubs them)
> - [^p5]: Parent exiting invoking state auto-terminates child session (working memory removed)
> - [^pR3]: Review-pass empirical: with `:src` pointing at an unregistered chart, the invocation processor queues `:error.platform` (type `:com.fulcrologic.statecharts/chart`) targeted at the parent's session. The parent is stuck unless it has a transition catching `:error.platform`.
