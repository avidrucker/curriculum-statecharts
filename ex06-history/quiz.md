# Quiz — ex06: History States (Shallow & Deep)

12 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 9 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. The `history` element lives where in a chart?

- A) At the chart root, alongside top-level states.
- B) Inside the compound state whose history it tracks, as one of its children.
- C) Inside every leaf state of the compound it tracks.
- D) In a separate `:histories` map on the chart options.

---

### 2. To invoke history restoration on entry, your transition's `:target` must be…

- A) The compound state's `:id`. The runtime auto-detects the history node inside.
- B) The history node's own `:id`. Targeting the compound directly enters its default initial — skipping history entirely.
- C) The keyword `:history`.
- D) A vector of `[compound-id history-id]`.

---

### 3. A `(history {:id :h :type :deep} :general)` declaration. What is `:general`?

- A) The compound state that owns the history node.
- B) The history node's parent.
- C) The history's **default target** — used the first time the chart enters via this history node (before any history exists to restore).
- D) The state to revert to if the history is corrupted.

---

### 4. The `history` element *requires* the default-target argument. What happens if you write `(history {:id :h :type :deep})` (no third arg)?

- A) The library silently defaults to the compound's first child.
- B) The chart loads, but history fails silently when invoked.
- C) The chart fails to construct. The `history` function throws `Wrong number of args (1)`. The default target is a positional argument, not optional.
- D) `:type :deep` overrides the requirement for a default.

---

### 5. When does the history node record the active state?

- A) On every transition the chart takes.
- B) When the runtime enters the compound parent.
- C) When the compound parent exits. The current active state (or full descent chain, for deep) at exit time becomes the recorded history. Earlier intermediate transitions update what *will* be recorded but don't snapshot independently.
- D) Only when a `(record-history!)` is explicitly invoked.

---

### 6. The chart enters `:settings → :privacy → :privacy-advanced`, then `:tab-general` (without leaving `:settings`), then `:close-settings`. Then `:open-settings` (via history). Where does the chart land?

- A) `:privacy-advanced` (history captured the earliest deep state).
- B) `:general`. History snapshots at exit; `:tab-general` was the last within-compound transition before `:close-settings`, so `:general` is what was recorded.
- C) `:privacy-basic` (privacy's default initial).
- D) `:main-screen` (no history available).

---

### 7. Shallow vs deep — what's the difference?

- A) Shallow restores only the immediate child of the history's compound parent. The chain below that child re-enters via the child's *own* default initials. Deep restores the full descent chain, including any nested compound's last-active child.
- B) Shallow is faster; deep is more accurate; otherwise they're interchangeable.
- C) Shallow doesn't record; deep records everything.
- D) Shallow only works for chart roots; deep only works inside parallel regions.

---

### 8. The exercise's chart has `:settings > :privacy > :privacy-advanced`. The history is `:type :shallow` with default `:general`. The user enters settings, navigates to `:privacy-advanced`, closes, re-opens. Where does the chart land?

- A) `:privacy-advanced` — same as deep, since `:privacy-advanced` was the last active state.
- B) `:privacy` — shallow restores `:privacy` (the immediate child of `:settings`); then `:privacy` enters at its default initial (`:privacy-basic`), not `:privacy-advanced`. The configuration includes `:privacy` and `:privacy-basic`; `:privacy-advanced` is NOT in it.
- C) `:general` — shallow falls back to the default target.
- D) `:main-screen` — shallow fails on multi-level navigation.

---

### 9. Tab-switching transitions (`:tab-general`, `:tab-privacy`, `:tab-notifications`) are declared on `:settings`, not on individual tabs. Why?

- A) Performance — declarations on the parent are faster.
- B) Ancestor walk (ex02). A transition on the parent fires whenever the chart is in any descendant of that parent. So `:tab-privacy` from `:notifications` works via the ancestor walk; from `:general` works; from `:privacy-advanced` works. Declaring on each tab separately would require triple-declaration with no semantic difference.
- C) Required by the `history` element.
- D) To prevent the transitions from competing with history.

---

### 10. `:app` is declared with `:initial :main-screen`. What does this do?

- A) Sets a fallback in case the chart can't be constructed.
- B) Overrides the document-order rule for picking `:app`'s initial child. Without `:initial`, `:settings` (the first child declared) would be the initial — opening the settings panel on chart startup, which isn't the intent. With `:initial :main-screen`, the chart starts at the main screen instead.
- C) Adds `:main-screen` to the chart's data model.
- D) Declares `:main-screen` as the "home" state for `:close-settings`.

---

### 11. The chart has been started but the user never opened settings. The history node `:settings-history` is targeted via `:open-settings`. What happens?

- A) Error — history requires a prior visit to record.
- B) The chart enters the **default target** declared on the history node (`:general` in the exercise). Default-target is exactly for this first-visit case.
- C) The chart enters `:settings` and lands at its document-order initial.
- D) The chart silently no-ops.

---

### 12. You add a history node to `:settings` but keep `:open-settings` targeting `:settings` (not the history node). Will history work?

- A) Yes — the runtime auto-detects the history node.
- B) Yes, but only for top-level navigation.
- C) No. History is invoked *only* when a transition targets the history node's `:id` directly. Targeting the compound state ignores the history node entirely and enters the compound's default initial every time. The history node exists in the chart but isn't consulted.
- D) Yes, but only if `:type :deep` is set.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 — history is a child element of the compound it tracks. |
| 2 | **B** | Tutorial Step 2 — target the history node's ID, not the compound. Verified in runbook Probe 3 (targeting compound skips history). |
| 3 | **C** | Tutorial Step 3 — default target for first-visit. |
| 4 | **C** | Empirically verified — `(history {...})` throws `Wrong number of args (1)`. Runbook "Try breaking it" #3. |
| 5 | **C** | Tutorial Step 4 + runbook Probe 4 — empirically verified that the exit-time snapshot is what matters. |
| 6 | **B** | Tutorial Step 4 + runbook Probe 4 — empirically verified. The intermediate `:tab-general` is what gets recorded; `:privacy-advanced` was overridden. |
| 7 | **A** | Tutorial Step 5 — shallow stops at the immediate child; deep records the chain. |
| 8 | **B** | Tutorial Step 5 + runbook Probe 2 — empirically verified shallow behavior. |
| 9 | **B** | Tutorial Step 6 — ancestor walk from ex02 applied to compound-level transitions. |
| 10 | **B** | Tutorial Step 7 — `:initial` overrides document order. The chart's document order would otherwise pick `:settings`. |
| 11 | **B** | Tutorial Step 3 — default target IS the first-visit behavior. |
| 12 | **C** | Tutorial Step 2 + runbook "Try breaking it" #1 — empirically verified that the history node must be the explicit target. |
