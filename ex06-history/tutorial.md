# Tutorial — ex06: History States (Shallow & Deep)

The conceptual narrative behind exercise 6. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex06 is dense — the chart has three levels of nesting and the new concept (history) interacts subtly with the existing rules from ex01–ex05. The exercise pays off: history is the right tool for "preserve where you were" UX patterns (modals, tabs, sub-flows) that would otherwise require manual save/restore.

---

## The story we're building

A settings panel. The user navigates between three tabs (`:general`, `:privacy`, `:notifications`), drills into a sub-detail (`:privacy-advanced` from `:privacy-basic`), closes the panel, and later re-opens it — and the chart **resumes them on the exact sub-state they were on**, not back at the default first tab.

Without history, the chart would forget. A standard transition like `:open-settings → :settings` enters `:settings` and lands at its document-order initial (`:general`) every time. Useful sometimes; wrong here. **History lets the chart remember.**

What you'll have grounded by the end:

1. The **`history` element** — its grammar, its required default-target, its `:type :shallow` vs `:type :deep` distinction.
2. **Targeting the history node** — why your transition's `:target` must be the *history node's* `:id`, not the compound parent's.
3. The **default target** — what fires the first time, before any history exists.
4. **When history records** — at exit time, regardless of how many within-compound transitions happened.
5. **Shallow vs deep** — shallow restores only the immediate child; deep restores the full descent chain. With nested children, the difference matters.

---

## Step 1 — A history node remembers where a compound state was

A compound state has children; exactly one is active at a time. When the chart leaves the compound (e.g., for a sibling state) and later re-enters, the runtime needs to pick a child to enter. **By default, this is the compound's initial state** — `:initial` keyword if set, else first document-order child. History gives you a different option: re-enter at *whichever child was active when the compound was last left*.

The element:

```clojure
(history {:id :settings-history :type :deep} :general)
```

Lives as a child of the compound state whose history it tracks. Three things to read here:

- **`:id`** — the history node's own ID. You'll target *this* ID from your transitions, not the parent compound's ID.
- **`:type`** — `:shallow` (default) or `:deep`. Determines how deeply the history is recorded.
- **The trailing argument** — the **default target**, used the very first time the chart enters via this history node (because there's no history yet). Required; the chart construction errors without it[^p6].

Per the library source, the element also accepts a transition-element form for the default — `(history {...} (transition {:target :general}))` — but the keyword shortcut is more common.

> **Common misconception** — "History is an option on the parent compound state."
>
> It isn't. History is a separate child element of the compound, with its own `:id`. The parent compound doesn't even know history exists; the history node is what your transitions explicitly target.

---

## Step 2 — Transitions target the *history node*, not the compound parent

This is the load-bearing distinction. Compare:

```clojure
;; Transition that ALWAYS enters at :settings's initial state
(transition {:event :open-settings :target :settings})

;; Transition that enters at the last-active descendant of :settings
(transition {:event :open-settings :target :settings-history})
```

The first ignores history — it enters `:settings` from "outside" and the runtime picks the default initial, regardless of any history recorded. **Verified empirically**[^p4]: targeting `:settings` directly even with a history node defined still lands at `:general` (the default).

The second consults `:settings-history`'s state: if any history exists, restore it; if not (first visit), use the default target keyword from the history declaration.

The pattern, in practice:

```clojure
(state {:id :app}
  (state {:id :settings}
    (history {:id :settings-history :type :deep} :general)
    ;; ... children ...
    )
  (state {:id :main-screen})
  (transition {:event :open-settings :target :settings-history})    ; targets the history node
  (transition {:event :close-settings :target :main-screen}))
```

> **Common misconception** — "I should target the compound state and the runtime will figure out history."
>
> It won't. History is an opt-in mechanism — the *only* way to invoke it is to target the history node's ID explicitly. A direct `:target :settings` skips history entirely.

---

## Step 3 — The default target fires the first time

The very first time the chart enters via the history node, there *is no history yet* — no compound exit has happened, so nothing was recorded. The history node falls back to its **default target** (the trailing argument or transition-element):

```clojure
(history {:id :sh :type :deep} :general)   ; default = :general
```

First `:open-settings` after `(t/start! env)`[^p2]:

```
config => #{:general :settings :app}
```

Chart lands at `:general` — the default. *Subsequent* `:open-settings` after a previous exit will use recorded history (covered next).

The default target is *required*. The library asserts on it at construction time; omitting it (or passing `nil`) throws an error[^p6]. So when you build a history node, decide upfront what should fire "on first ever visit."

> **Common misconception** — "If history's empty, the chart enters the compound's default initial state."
>
> No — the history node's own default fires. These often align (you make the history default the same as the compound's initial), but they're separate decisions. The history's default is whatever you put as the third argument to `history`.

---

## Step 4 — History records at exit time

When does the history node "snapshot" the active state? **At the moment the chart exits the compound parent.** Within-compound transitions update *what will be recorded* but don't cause separate snapshots.

Empirical proof[^p5]: drive the chart through `:open-settings → :tab-privacy → :privacy-detail → :tab-general`, then `:close-settings → :open-settings`. The chart lands at `:general` — the most recent within-compound state. Earlier visits to `:privacy-advanced` are *not* remembered; only the state right before exit matters.

This matches the SCXML standard semantics: history captures the active state when the parent compound exits. The runtime then stores that under the history node's `:id` in the working memory, and restores it on the next history-targeted entry.

---

## Step 5 — Shallow vs deep

The distinction matters when the compound has *nested* compound children. The exercise's chart:

```
:settings
  ├─ :general                  (atomic)
  ├─ :privacy                  (compound)
  │    ├─ :privacy-basic       (atomic)
  │    └─ :privacy-advanced    (atomic)
  └─ :notifications            (atomic)
```

The user navigates from `:settings/general` → `:settings/privacy/privacy-advanced`, then closes settings.

| History type | What's recorded | What's restored |
| --- | --- | --- |
| `:shallow` | `:privacy` (the immediate child of `:settings`) | `:privacy` — but `:privacy` is itself compound, so its *default initial* (`:privacy-basic`) becomes the active leaf[^p3] |
| `:deep` | The whole chain: `:privacy > :privacy-advanced` | `:privacy-advanced` directly[^p2] |

So shallow + nested children = "back to the right tab but lose the sub-state"; deep = "back to exactly where you were."

For the exercise's chart, `:type :deep` is required — the test asserts `(t/in? env :privacy-advanced)` after re-open. Shallow would land at `:privacy-basic` and the assertion would fail.

> **Common misconception** — "Shallow is faster / cheaper / more common."
>
> Neither type has runtime cost differences worth thinking about. Choose based on intent: **shallow** for "remember which tab," **deep** for "remember the exact spot." For most multi-level UIs, deep is what you want; for top-level-only navigation, shallow is fine.

---

## Step 6 — Tab-switching transitions live on the parent

The chart's `:tab-general`, `:tab-privacy`, `:tab-notifications` transitions all live on **`:settings` itself**, not on each tab:

```clojure
(state {:id :settings}
  (history {:id :settings-history :type :deep} :general)
  (transition {:event :tab-general :target :general})
  (transition {:event :tab-privacy :target :privacy})
  (transition {:event :tab-notifications :target :notifications})
  (state {:id :general})
  (state {:id :privacy} ...)
  (state {:id :notifications}))
```

This is the **ancestor walk** pattern from ex02 — a transition declared on the parent compound fires whenever the chart is in any of its children. So `:tab-privacy` from `:notifications` works (ancestor walk finds the transition on `:settings`); from `:general` works; from `:privacy-advanced` (deep child) works.

The alternative would be declaring `:tab-privacy`, `:tab-general`, `:tab-notifications` separately on each of the three tab states — three triple-declarations of the same transitions. The parent-level placement is much terser and the right idiom for "this event always means the same thing regardless of current sub-state."

> **Common misconception** — "If a tab is the source, the transition has to be inside that tab."
>
> The transition is declared on whatever state defines the *scope* in which it should fire. Putting `:tab-privacy` on `:settings` makes it fire from anywhere inside `:settings`. Putting it on `:general` would limit it to firing only when `:general` is active.

---

## Step 7 — Putting it together

The chart, in plain English:

> An app has a main screen and a settings panel. The settings panel has three tabs and a sub-detail inside `:privacy`. The user can open settings, navigate tabs, drill into details, close back to main, re-open — and the chart restores their exact prior state (deep history).

The chart, as a tree:

```
:app (initial :main-screen)
  ├─ on :open-settings → :settings-history
  ├─ on :close-settings → :main-screen
  ├─ :settings
  │    ├─ <history :settings-history, deep, default :general>
  │    ├─ on :tab-general → :general
  │    ├─ on :tab-privacy → :privacy
  │    ├─ on :tab-notifications → :notifications
  │    ├─ :general
  │    ├─ :privacy
  │    │    ├─ :privacy-basic → :privacy-advanced on :privacy-detail
  │    │    └─ :privacy-advanced
  │    └─ :notifications
  └─ :main-screen
```

The chart, as Clojure (the solution):

```clojure
(def settings-panel
  (statechart {}
    (state {:id :app :initial :main-screen}
      (state {:id :settings}
        (history {:id :settings-history :type :deep} :general)
        (transition {:event :tab-general :target :general})
        (transition {:event :tab-privacy :target :privacy})
        (transition {:event :tab-notifications :target :notifications})
        (state {:id :general})
        (state {:id :privacy}
          (state {:id :privacy-basic}
            (transition {:event :privacy-detail :target :privacy-advanced}))
          (state {:id :privacy-advanced}))
        (state {:id :notifications}))
      (state {:id :main-screen})
      (transition {:event :open-settings :target :settings-history})
      (transition {:event :close-settings :target :main-screen}))))
```

Notice the chart uses **`:initial :main-screen` on `:app`** to override document order — `:settings` is declared first but `:main-screen` is the initial. (Document order would have picked `:settings`, opening the panel on startup, which isn't the intent.)

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why the transition targets `:settings-history` and not `:settings`.** Targeting the parent ignores history; targeting the history node consults it.
- **The difference between the history's default target and the compound's initial.** The history's default is what fires on first-ever entry via history. The compound's initial is what fires when entered *without* going through history.
- **When history records.** At the moment the compound exits — capturing whatever was active just before. Within-compound transitions update what will be recorded.
- **Why this chart needs deep history.** The test asserts re-opening to `:privacy-advanced` (a nested leaf). Shallow would only restore `:privacy` and land at `:privacy-basic`.
- **Where the tab-switching transitions live and why.** On `:settings` (the parent), so they fire from any tab. The ancestor walk does the rest.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns.

---

> **Verified empirically (probes run during ex06 authoring):**
>
> - [^p2]: Full solution end-to-end — deep history correctly restores `:privacy-advanced` after close + re-open
> - [^p3]: Shallow history with the same chart structure restores `:privacy` but lands at `:privacy-basic` (default initial of `:privacy`)
> - [^p4]: Targeting `:settings` directly (not the history node) lands at `:general` regardless of recorded history
> - [^p5]: History snapshots at exit time — within-compound transitions update what will be recorded, intermediate visits don't matter
> - [^p6]: `history` element requires a default target — omitting it throws `Wrong number of args (1)` at chart construction
