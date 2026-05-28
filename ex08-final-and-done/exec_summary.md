# Executive Summary — ex08: Final States, `done.state` Events, and Chart Exit

The 5-minute refresher. Use this when you've completed ex08 once and want to re-anchor.

---

## The one big idea

**Entering a `final` element auto-fires `done.state.<parent-id>`** — a synthesized internal event the parent compound state can handle to react. For parallels, `done.state.<parallel-id>` fires only when *all* regions are final. **Top-level finals terminate the chart entirely** (configuration empty, `running? false`).

Three mechanics in one exercise: terminal states, auto-events, and chart exit.

---

## What we built

A processing pipeline. Two parallel tasks; when both finish, advance to `:complete`; then `:finalize` exits the chart.

```clojure
(def processing-pipeline
  (statechart {}
    (state {:id :pipeline}
      (parallel {:id :processing}
        (state {:id :validation}
          (state {:id :validating}
            (transition {:event :validation-ok :target :validated}))
          (final {:id :validated}))                                   ; region-level final
        (state {:id :enrichment}
          (state {:id :enriching}
            (transition {:event :enrichment-ok :target :enriched}))
          (final {:id :enriched})))                                   ; region-level final
      (state {:id :complete}
        (transition {:event :finalize :target :finished}))
      (transition {:event :done.state.processing :target :complete}))   ; auto-event handler
    (final {:id :finished})))                                          ; top-level final = chart exit
```

Two layers of finals: per-region finals signal "this task done"; top-level final signals "chart done."

---

## Mechanics in 30 seconds

- **`(final {:id :x})`** = a terminal state. `{:node-type :final}`. No children allowed (transitions inside throw at construction).
- **Auto-event fires on final entry:** `(t/run-events! env :go)` → chart enters a final → runtime synthesizes `done.state.<parent-id>` and adds it to the internal event queue.
- **Parallel completion:** `done.state.<parallel-id>` fires only when ALL regions are final. Partial completion fires per-region `done.state.<region-id>` events but not the parallel's.
- **Auto-event shape:** `{:type :internal, :name :done.state.<parent-id>, :sendid <final-id>, :data {} ...}`. Just a regular event from the matching perspective.
- **Top-level final = chart exit.** Configuration becomes `#{}`, `running?` becomes `false`. Subsequent events are silently dropped.
- **Stuck vs exited:** `(running? env)` is the diagnostic. False = exited; True + non-empty config = stuck at a leaf with no transitions.

---

## Composes with

- **ex04 (parallel)** — `done.state.<parallel-id>` is the auto-event for parallels; relies on the all-regions-final rule.
- **ex06 (history)** — finals don't record history; they're terminal.
- **ex09 (invocations)** — `done.invoke.<invokeid>` is the analogous auto-event for child charts; same mechanism, different prefix.
- **ex02's string-prefix matching footgun** — bites here: a transition listening for `:done` catches every `done.state.<id>` and `done.invoke.<id>` event the runtime synthesizes.

---

## Gotchas to remember

- **Finals can't have children.** `(final {:id :x} (transition ...))` throws `Illegal children of :final` at construction.
- **`done.state.<parallel>` fires only when ALL regions are final** — not on partial completion. For "first completion wins," listen for per-region `done.state.<region>` events instead.
- **Top-level final is the only kind that exits the chart.** A final inside a compound fires `done.state.<parent>` but the chart keeps running.
- **`:done` as a transition's `:event` accidentally catches every `done.state.X` and `done.invoke.X` event** via ex02's string-prefix matching fallback. Use full names.
- **After chart exit, events don't error — they're silently dropped.** `(running? env)` is the only reliable "is chart alive?" check.
- **`done-data` (optional element)** can attach data to the auto-event. Not used in this exercise; useful for parallel-completing aggregations.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex08-final-and-done :as ex08])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex08/processing-pipeline
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/in? env :validating)               ; → true
(t/in? env :enriching)                ; → true   (parallel)

;; Complete one region — done.state.processing does NOT fire yet
(t/run-events! env :validation-ok)
(t/in? env :validated)                ; → true   (region's final)
(t/in? env :complete)                 ; → false  (parallel not all done)

;; Complete the other — NOW done.state.processing fires
(t/run-events! env :enrichment-ok)
(t/in? env :complete)                 ; → true   (auto-event handled)

;; Finalize — top-level final exits the chart
(t/run-events! env :finalize)
;; (running? env) would be false now; config is #{}
```

If those five checks pass, ex08 is solid. Move on to [ex09 (invocations)](../ex09-invocations/) — the final, densest module: child chart invocations, separate sessions, manual orchestration.
