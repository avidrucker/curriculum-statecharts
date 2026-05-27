# Quiz — ex07: Internal vs External Transitions

11 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 8 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. When you omit `:type` on a `transition` element, what's the default?

- A) `:internal`.
- B) `:external`. Every transition without an explicit `:type` is external. Most exercises in this curriculum use default transitions; ex07 is the first where the distinction matters.
- C) `:default` (a distinct third type).
- D) Depends on whether the target is a descendant of the source.

---

### 2. A transition declared on `:editor` (compound) targets `:editing` (a descendant of `:editor`). With `:type :external`, what happens to `:editor` when the transition fires?

- A) Nothing — `:editor` stays active throughout.
- B) `:editor` exits and re-enters. Its on-exit fires, then its on-entry fires again. The exit set computed for an external transition declared on a compound state with a descendant target includes the compound itself.
- C) `:editor` exits but doesn't re-enter; the chart stays in the descendant only.
- D) An error — external transitions can't target descendants.

---

### 3. The same transition with `:type :internal`. What happens to `:editor`?

- A) `:editor` is exited and re-entered, but more efficiently.
- B) `:editor` stays in the configuration throughout. Its on-exit and on-entry do *not* fire. Only the active leaf changes (from `:viewing` to `:editing`).
- C) `:editor` exits but the runtime defers re-entry until the next event.
- D) An error — internal transitions require a `:cond`.

---

### 4. When is `:type :internal` meaningful (i.e., changes behavior from external)?

- A) Always — every transition can benefit from internal.
- B) Only when the transition has no `:event`.
- C) **Only when the source state is a compound state AND the target is a strict descendant of that source.** In every other case (non-descendant target, child-declared transition, self-target), internal behaves the same as external.
- D) Only inside parallel regions.

---

### 5. A transition has `:type :internal` and targets a state *outside* the source's descendant chain. What does the library do?

- A) Errors at chart construction.
- B) Treats it as if you wrote `:type :external`. The annotation is silently ignored; the source state still exits, side effects fire. No warning.
- C) Errors at runtime when the transition fires.
- D) Splits the transition into two — one internal, one external.

---

### 6. A transition declared on `:viewing` (a child of `:editor`) targets `:editing` (sibling of `:viewing`). With `:type :external`, what happens to `:editor`?

- A) `:editor` exits and re-enters.
- B) `:editor` stays — its on-entry/on-exit do *not* fire. The LCA of `:viewing` and `:editing` is `:editor`, and external transitions don't exit the LCA itself. `:type :internal` here would be redundant.
- C) Same behavior as if `:type :internal` were set on a transition declared on `:editor` with the same target.
- D) Errors — sibling-to-sibling transitions aren't allowed.

---

### 7. The exercise's chart uses `:type :internal` on `:edit` and `:preview` but NOT on `:done`. Why?

- A) Because `:done` is declared after the others.
- B) Because `:done`'s target (`:closed`) is *outside* `:editor`. The transition needs to actually exit `:editor` so `save-doc!` fires. Marking `:done` internal would be silently ignored (target isn't a descendant), but the intent reads more clearly with the default external.
- C) Because `:done` carries data and only external transitions support that.
- D) Because internal transitions can't have ancestor-walk semantics.

---

### 8. A transition is declared `(transition {:event :ping :target :s :type :internal})` where `:s` is the same state (self-target). What happens on `:ping`?

- A) Nothing — internal self-targets are no-ops.
- B) The state exits and re-enters, firing on-exit and on-entry. The `:type :internal` is silently ignored for self-target transitions (the runtime treats source = target as external).
- C) The state stays without firing on-entry/on-exit, but the transition's body still runs.
- D) An error — self-targets require `:cond`.

---

### 9. You want a state to update its data on an event *without* re-firing its on-entry. Which mechanism should you use?

- A) `:type :internal` on a self-target transition.
- B) A **targetless transition** (from ex03b, often via the `handle` convenience helper). The transition has no `:target` at all; the state's body just runs without any state change.
- C) `:type :external` with a guard that returns false.
- D) Splitting the state into two.

---

### 10. Block 2 of the exercise's test asserts `(= 1 @load-count)` after `:edit` fires. What does this assertion *prove*?

- A) That `:edit` is declared in the chart.
- B) That `:edit` fired without re-running `:editor`'s on-entry. If on-entry had re-fired, `load-count` would be 2. The test is specifically checking that the transition was internal (didn't exit/re-enter the compound).
- C) That `:edit` is the first event handler.
- D) That `:edit` has no script body.

---

### 11. Block 4 asserts `(= 1 @save-count)` after `:done`. What does this assertion *prove*?

- A) That `:done` was the last event fired.
- B) That `:done` was external (or that its target `:closed` was outside `:editor`). The transition's effect on the configuration triggered `:editor`'s on-exit, which called `save-doc!`. If `:done` had been declared in a way that didn't exit `:editor`, save-count would still be 0.
- C) That `save-doc!` was called from the test harness, not the chart.
- D) That the chart has reached a terminal state.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 + runbook Probe 3 (default behavior matches `:external` explicitly). |
| 2 | **B** | Tutorial Step 2 — external on a compound with descendant target exits the compound. |
| 3 | **B** | Tutorial Step 3 — internal keeps the compound active. |
| 4 | **C** | Tutorial Step 4 — narrow applicability of internal. |
| 5 | **B** | Tutorial Step 5 + runbook Probe 3 — empirically verified the silent no-op behavior. |
| 6 | **B** | Tutorial Step 6 — LCA-based exit-set computation; child-declared transitions don't exit the parent regardless of `:type`. |
| 7 | **B** | Tutorial Step 3 (the editor design rationale). |
| 8 | **B** | Empirically verified (runbook Probe 4) — internal self-target re-enters. |
| 9 | **B** | ex03b's `handle` (or raw targetless `transition` element) is the right tool for "update data, stay put without re-entering." `:type :internal` self-target doesn't achieve this. |
| 10 | **B** | The exact pedagogical purpose of the assertion — the counter is the proof that on-entry didn't re-fire. |
| 11 | **B** | The other side of the same pedagogy — save-count is the proof that on-exit *did* fire. |
