# Glossary — ex06: History States (Shallow & Deep)

Terms introduced (or first deeply covered) in ex06. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex01–ex05 covered states, transitions, configurations, parallels, multi-target. This file adds what ex06 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `history` (element)  *(criticality: 9)*

The element from `com.fulcrologic.statecharts.elements` that creates a **history pseudo-state**: a marker inside a compound state that records the compound's last-active descendant on exit, and restores it on re-entry (when targeted by ID).

Call signature:

```clojure
(history {:keys [id type deep?] :as attrs} default-transition)
```

- **`:id`** (required, by convention) — the history node's own keyword ID. Used as the `:target` of transitions that want to invoke history restoration.
- **`:type`** — `:shallow` (default) or `:deep`. Determines what the history records and restores.
- **`:deep?`** — alias for `:type :deep` (per the library source). `(history {:id :h :deep? true} :general)` ≡ `(history {:id :h :type :deep} :general)`.
- **`default-transition`** (required positional argument) — either a transition element `(transition {:target :foo})` or the shortcut: just the target keyword `:foo`. The library wraps a bare keyword in a `transition` element internally.

Lives as a child of the compound state whose history it tracks. The compound state itself doesn't reference the history node by name — it just contains it.

### `:type :deep` vs `:type :shallow`  *(criticality: 8)*

The choice of how *deeply* the history records and restores.

| Type | Records | Restores |
| --- | --- | --- |
| `:shallow` | The immediate child of the history's parent compound that was active at exit | The immediate child re-enters at *its* default initial |
| `:deep` | The full chain of active states all the way to the atomic leaf | The full chain — including any nested compound's last-active descendant[^p3] |

For a compound parent with no nested compound children, the two are equivalent. For multi-level navigation (e.g., `:settings > :privacy > :privacy-advanced`), only `:deep` restores `:privacy-advanced` directly; `:shallow` restores `:privacy` and re-enters at `:privacy-basic`.

For this exercise, `:type :deep` is required by the test. For top-level-only tab navigation, `:shallow` suffices.

### default target (of a history node)  *(criticality: 8)*

The third positional argument to `(history ...)` — the state to enter the **first time** the chart targets the history node, before any history has been recorded.

```clojure
(history {:id :sh :type :deep} :general)
;;                              ^^^^^^^^
;;                              default target — used when no history exists yet
```

Required at construction time[^p6]. The library asserts the target is non-nil and there's no `:cond` or `:event` on it (default history transitions must be unconditional and event-less).

Distinct from the compound's `:initial`:
- The compound's `:initial` fires when the compound is entered *without* going through history.
- The history's default fires when the compound is entered *through* history *and there's no recorded state yet*.

These often align (the compound's initial and the history's default are the same state), but they're independent decisions.

### history target (transition's `:target`)  *(criticality: 8)*

To invoke history restoration, a transition must target the **history node's `:id`**, not the compound parent's:

```clojure
;; Skips history — enters :settings's default initial every time
(transition {:event :open-settings :target :settings})

;; Invokes history — restores last-active or uses default target
(transition {:event :open-settings :target :settings-history})
```

The library does *not* auto-detect history when you target the compound; history is opt-in via explicit targeting[^p4].

### history snapshot timing  *(criticality: 7)*

History records the active state(s) **at exit time**, not on every transition. As the chart moves through within-compound states, the *eventual* recorded state updates — but no separate snapshot is taken until the compound actually exits[^p5].

```
:open-settings → :tab-privacy → :privacy-detail → :tab-general → :close-settings → :open-settings
;;                                                                                  ^
;; History recorded as :general (the last within-compound state before exit), NOT :privacy-advanced
```

If you want to record a snapshot mid-flight, the chart provides no built-in mechanism — history is exit-triggered only.

### ancestor-level transition (revisited)  *(criticality: 6)*

ex02 established the rule: a transition declared on a state fires when that state (or any of its descendants) is active. ex06 leans on this for the tab-switching pattern:

```clojure
(state {:id :settings}
  (transition {:event :tab-privacy :target :privacy})    ; on the PARENT
  (state {:id :general})                                  ; tabs are children
  (state {:id :privacy} ...)
  (state {:id :notifications}))
```

`:tab-privacy` fires whenever `:settings` is active — which includes when `:general`, `:notifications`, or any descendant of `:privacy` is the active leaf. One declaration covers all the tabs without per-tab duplication.

### `:initial` on a state  *(criticality: 6)*

Optional attribute on a `state` (or `parallel`) that overrides the default document-order initial. Used to specify exactly which child should be entered when the parent is entered:

```clojure
(state {:id :app :initial :main-screen}     ; → :main-screen, not document-order first
  (state {:id :settings} ...)               ; declared first, but NOT the initial
  (state {:id :main-screen}))
```

Without `:initial`, the document-order rule (from ex01) picks the first declared child. ex06 is the first exercise where `:initial` is necessary — `:settings` is naturally declared before `:main-screen` (the history node's container should come first conceptually), but the chart should start at `:main-screen` (the user's "home").

### history pseudo-state  *(criticality: 3)*

In SCXML terminology, history is a **pseudo-state** — a chart element that *looks like* a state (has an `:id`, can be targeted by transitions) but isn't a state the chart can be "in." After history fires, the chart enters the restored (or default) target; the history node itself doesn't appear in the configuration set.

`(t/in? env :settings-history)` returns `false` — even after `:open-settings` invokes it. The history node is purely a routing mechanism.

This is distinct from the "transient state" pattern of `choice` (ex03c) — choice states *are* real states (they enter and exit, possibly briefly). History pseudo-states don't enter at all; they redirect.

---

> **Verified empirically (probes run during ex06 authoring):**
>
> - [^p3]: Shallow history restores `:privacy` (the immediate child) but lands at `:privacy-basic` (default initial)
> - [^p4]: Targeting compound directly (skipping history) lands at default initial regardless of recorded history
> - [^p5]: History snapshots at exit time, not on every transition
> - [^p6]: `history` element requires a default target — omitting it throws at construction
