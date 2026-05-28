# Executive Summary — ex09: Child Chart Invocations

The 5-minute refresher. Use this when you've completed ex09 once and want to re-anchor.

ex09 is the densest module — and the curriculum's payoff: how charts COMPOSE.

---

## The one big idea

**`invoke` spawns a separate child statechart session, alive for the duration of the parent state containing the invoke.** The child has its own session-id (= the invoke's `:id`), its own working memory, its own configuration. When the child reaches its top-level final, the runtime auto-fires `done.invoke.<invokeid>` back at the parent.

Three concrete consequences:

- Parent and child are **two sessions** in the working-memory-store, communicating through events.
- The child is **lifecycle-coupled** to the parent's state — parent leaves → child auto-terminated.
- `t/new-testing-env` **doesn't actually run child charts** (`MockInvocations` stubs them). The test must use `simple/simple-env` + manual protocol calls.

---

## What we built

A checkout flow that delegates payment to a separate chart. When the payment chart finishes (its `:charged` final), the parent advances to `:confirmation`.

```clojure
;; Provided:
(def payment-chart
  (statechart {}
    (state {:id :charging} (transition {:event :charge-complete :target :charged}))
    (final {:id :charged})))                                      ; top-level final in CHILD

;; Built:
(def checkout-chart
  (statechart {}
    (state {:id :checkout}
      (state {:id :collecting-info}
        (transition {:event :pay :target :processing-payment}))
      (state {:id :processing-payment}
        (invoke {:id :payment-process :type :statechart :src :payment-chart})    ; spawn child
        (transition {:event :done.invoke.payment-process :target :confirmation}))
      (state {:id :confirmation}
        (transition {:event :finish :target :done}))
      (final {:id :done}))))
```

The pattern that ex09 establishes: parent + child charts that communicate via the runtime's event-routing infrastructure.

---

## Mechanics in 30 seconds

- **`(invoke {:id :pp :type :statechart :src :payment-chart})`** — declared inside the state where the child should run. `:id` becomes the child session-id. `:src` is a registry key.
- **Register both charts:** `(simple/register! env :checkout-chart parent)` and `(simple/register! env :payment-chart child)`. Both required before `(sp/start! ...)`.
- **`simple/simple-env`** has the real invocation processor. `t/new-testing-env` has `MockInvocations` and doesn't actually run children.
- **Child session = invokeid.** `(sp/get-working-memory wmstore env :payment-process)` returns the child's wmem.
- **Auto-event on child final:** `done.invoke.<invokeid>` with `:target` set to the *parent's* session-id, `:invokeid` and `:source-session-id` recording the child.
- **Parent leaves invoking state → child auto-terminated.** Working memory removed from store. No manual cleanup needed.
- **Manual orchestration in tests:** `sp/process-event!` + `sp/save-working-memory!` (per session) + `sp/receive-events!` for routing the auto-event between sessions.

---

## Composes with

- **ex01's chart-is-data** — the parent and child are both just data; the registry holds them as values.
- **ex08's `done.state` / final patterns** — `done.invoke.<id>` is the invocation analog of `done.state.<id>`.
- **ex06 (history)** — each child session is a separate chart with its own working memory, so it has its own history bookkeeping. The library's design implies history-inside-invoked-chart should work straightforwardly, but the curriculum doesn't have an explicit probe for this (see top-level `issues.md` verification queue).
- **ex02's string-prefix matching footgun** — bites for the 4th time: `:done.invoke.payment` prefix-matches `:done.invoke.payment-process`. Don't name invokeids that string-prefix each other.

---

## Gotchas to remember

- **`t/new-testing-env` doesn't actually run children** (MockInvocations stubs them). For invocation-bearing charts, use `simple/simple-env` and manual protocol calls.
- **Forgetting to register the child** queues an `:error.platform` event. The child never starts; the parent doesn't progress. Catch the error explicitly or remember to register.
- **`:src` is a REGISTRY KEY, not a chart value.** Pass `:payment-chart` (the key), not the chart map itself.
- **Done-event's `:target` is the PARENT's session-id** — addressing info. The `:invokeid` field has the child's id (= the invoke's `:id`).
- **String-prefix matching for invokeid names.** `:payment` and `:payment-process` will cross-match. Pick names that are full-segment-distinct.
- **Parent and child are siblings in the working-memory-store**, not nested. The relationship lives in the chart structure, not the storage layout.
- **`:autoforward true`** is rarely what you want. Forwards every parent event to the child — useful for "child wraps parent" semantics, surprising in most other cases.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex09-invocations :as ex09])
(require '[com.fulcrologic.statecharts :as sc])
(require '[com.fulcrologic.statecharts.events :refer [new-event]])
(require '[com.fulcrologic.statecharts.protocols :as sp])
(require '[com.fulcrologic.statecharts.simple :as simple])

(def env (simple/simple-env))
(simple/register! env :checkout-chart ex09/checkout-chart)
(simple/register! env :payment-chart ex09/payment-chart)

(let [s0 (sp/start! (::sc/processor env) env :checkout-chart {::sc/session-id :s1})]
  (sp/save-working-memory! (::sc/working-memory-store env) env :s1 s0))

;; Drive parent to :processing-payment (child spawns)
(let [wm (sp/get-working-memory (::sc/working-memory-store env) env :s1)
      wm' (sp/process-event! (::sc/processor env) env wm (new-event :pay))]
  (sp/save-working-memory! (::sc/working-memory-store env) env :s1 wm'))

;; Child exists with its own config
(::sc/configuration (sp/get-working-memory (::sc/working-memory-store env) env :payment-process))
;; → #{:charging}
```

If you can see the child's `#{:charging}` config separate from the parent's, ex09 is solid. You've completed the curriculum.

---

## What you can now build

The curriculum's payoff. With ex01–ex09 internalized, you can model:

- **Multi-step flows** (login, checkout, wizards) using compound states + guards
- **Concurrent independent state** (UI modes, background tasks) using `parallel`
- **Resume-on-reentry navigation** (tabs, modals) using history
- **Composed multi-chart apps** (routing → feature → worker) using `invoke`
- **Lifecycle-coupled work** (modal open = start; close = cleanup) using `on-entry` / `on-exit`

This is the SCXML feature set Tony Kay's library follows. From here, the next learning surface is **Fulcro integration** — using charts to manage UI state, routes, and async work in a real Fulcro app.
