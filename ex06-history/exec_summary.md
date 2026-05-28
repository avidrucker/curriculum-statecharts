# Executive Summary — ex06: History States (Shallow & Deep)

The 5-minute refresher. Use this when you've completed ex06 once and want to re-anchor.

ex06 is dense — three levels of nesting and a new pseudo-state.

---

## The one big idea

**A `history` element inside a compound state records the compound's last-active descendant on exit and restores it on re-entry — *if* the re-entering transition targets the history node's `:id` (not the compound's).**

Two distinctions matter:

- **Shallow vs deep:** shallow records only the immediate child of the compound; deep records the full descent chain (including nested compound children's leaves).
- **Target the history node, not the parent:** targeting the parent compound directly ignores history and enters the default initial. The runtime opts into history only when targeted by the history node's `:id` explicitly.

---

## What we built

A settings panel with three tabs (one with a nested sub-state). Closing and re-opening restores the exact sub-state via deep history.

```clojure
(def settings-panel
  (statechart {}
    (state {:id :app :initial :main-screen}
      (state {:id :settings}
        (history {:id :settings-history :type :deep} :general)        ; deep history, default :general
        (transition {:event :tab-general       :target :general})
        (transition {:event :tab-privacy       :target :privacy})
        (transition {:event :tab-notifications :target :notifications})
        (state {:id :general})
        (state {:id :privacy}
          (state {:id :privacy-basic}
            (transition {:event :privacy-detail :target :privacy-advanced}))
          (state {:id :privacy-advanced}))
        (state {:id :notifications}))
      (state {:id :main-screen})
      (transition {:event :open-settings  :target :settings-history})   ; target HISTORY NODE
      (transition {:event :close-settings :target :main-screen}))))
```

The settings panel is a 3-level nested chart. After visiting `:privacy-advanced` and closing, re-opening restores `:privacy-advanced` directly via deep history.

---

## Mechanics in 30 seconds

- **`(history {:id :h :type :deep} :default-target)`** — declares a history pseudo-state. `:type :shallow` or `:type :deep`. Third arg is the default target (used on first visit before any history is recorded). Required.
- **Target the history node's `:id`** to invoke restoration. `(transition {:event :open-settings :target :settings-history})`. Targeting the compound's `:id` directly ignores history.
- **History records at *exit* time** — when the compound parent is exited, the current active descendant becomes the recorded value. Within-compound transitions update what *will* be recorded but don't snapshot independently.
- **Shallow:** restores the immediate child of the history's parent; the child then enters via its own default initial.
- **Deep:** restores the full chain — including nested compound state's last-active descendant.
- **`(t/in? env :history-id)` returns `false` always** — history is a pseudo-state. The runtime resolves history to a real target before settling; the history node itself never appears in the configuration.
- **`:initial` on a state overrides document order.** `(state {:id :app :initial :main-screen} ...)` — without this, document order picks `:settings` (declared first) as the initial, which isn't the intent.
- **Tab-switching transitions live on the parent compound,** not on individual tabs. Ancestor walk from ex02 makes them fire from any tab.

---

## Composes with

- **ex01's chart-root and `:initial`** — ex06 needs `:initial` on `:app` to override document order.
- **ex02's ancestor walk** — tab-switching transitions on the parent compound work via this rule.
- **ex04 (parallel)** — history works inside parallel regions too; each region tracks its own (verified empirically).
- **ex05's multi-target** — combined with history, you can restore multiple regions' last-active leaves via `:target [:hist1 :hist2]`.

---

## Gotchas to remember

- **Targeting the compound directly skips history.** This is the most common "why isn't history working?" bug. The transition's target MUST be the history node's `:id`.
- **Shallow loses sub-state in nested compounds.** If a tab has its own sub-states, shallow restores the tab but lands at the tab's default initial — not the actual sub-state. Deep is what you want for multi-level navigation.
- **`history` requires a default target.** `(history {:id :h :type :deep})` (no third arg) throws `Wrong number of args (1)` at construction.
- **History records at exit, not on every transition.** Earlier intermediate visits within the compound are overwritten by whatever was active just before the parent compound exited.
- **Event-name prefix collision** — if your chart has both `:enter` (target the compound directly) and `:enter-h` (target the history node) transitions, the runtime fires `:enter` for an incoming `:enter-h` event because `:enter` is a string-prefix of `:enter-h`. ex02's string-prefix matching footgun strikes again — see [ex06 gotchas #7](./gotchas.md).
- **`goto-configuration!` preserves history values** (verified in the v1 curriculum review pass — see [ex03/issues.md "Resolved"](../ex03-guards/issues.md#resolved)). The library docstring claim holds.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex06-history :as ex06])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex06/settings-panel
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/in? env :main-screen)                  ; → true (initial)

;; First open uses default :general
(t/run-events! env :open-settings)
(t/in? env :general)                      ; → true (history default on first visit)

;; Navigate to deep state
(t/run-events! env :tab-privacy :privacy-detail)
(t/in? env :privacy-advanced)             ; → true

;; Close and re-open — deep history restores :privacy-advanced
(t/run-events! env :close-settings :open-settings)
(t/in? env :privacy-advanced)             ; → true (deep history!)
```

If those four checks pass, ex06 is solid. Move on to [ex07 (internal transitions)](../ex07-internal-transitions/) — the mechanism for transitions inside a compound state that DON'T re-fire its on-entry/on-exit.
