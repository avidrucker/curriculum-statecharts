# Quiz — ex05: Multi-Target Transitions

10 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 7 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. What does `:target [:stopped :unmuted]` express on a transition?

- A) Two alternative targets — the runtime picks one based on document order.
- B) A *sequence* of targets — the chart enters `:stopped`, then exits, then enters `:unmuted`.
- C) Multiple simultaneously-entered atomic states. When the transition fires, both `:stopped` and `:unmuted` become active in the configuration (each in their respective region of a parallel).
- D) An OR condition — the chart enters whichever of the two states is reachable from the current configuration.

---

### 2. The exercise's chart could be solved with `:target :stopped` (single keyword, no vector). Why?

- A) Because the library auto-completes single-targets by expanding them into multi-targets.
- B) Because `:unmuted` is the default-document-order initial of `:volume`. When the transition re-enters `:player`, `:playback` lands in the explicit `:stopped`, and `:volume` defaults to `:unmuted`. Both `(t/in? env :stopped)` and `(t/in? env :unmuted)` pass.
- C) Because the test uses `(t/in? env :unmuted)` as a soft check that doesn't actually fail.
- D) It can't — the test would fail with single-target.

---

### 3. When *is* multi-target genuinely necessary?

- A) Always, in any chart with a `parallel`.
- B) Never — single-target plus default initials covers every case.
- C) When you want to land the chart in a **non-default initial** in some region. If `:volume` had `:muted` declared first (making it the default), but you want `:reset` to land in `:unmuted`, you'd need `:target [:stopped :unmuted]` to override the default.
- D) Only when the transition is declared on the parallel itself.

---

### 4. Where is the `:reset` transition placed in the exercise's solution?

- A) Inside each region's atomic states (broadcast pattern).
- B) Inside one of `:playback`'s children only (e.g., on `:playing`).
- C) As a direct child of the `parallel` element. This means the transition is "active" whenever the parallel is in the configuration — regardless of which atomic leaves each region happens to be in.
- D) At the chart root (sibling of the parallel).

---

### 5. `:target [:stopped]` (vector with one element) is equivalent to…

- A) `:target nil` (no target).
- B) `:target :stopped` (single keyword). The library normalizes single-keyword targets to one-element vectors internally; the two forms are indistinguishable to the runtime.
- C) A targetless transition.
- D) An error — multi-target requires at least two targets.

---

### 6. The order of targets in the vector — `[:stopped :unmuted]` vs `[:unmuted :stopped]`. Does it matter?

- A) Yes — the first target is entered first; the second after.
- B) Yes — the first target's region is given priority for entry actions.
- C) No. The library resolves each target to its region in the chart's structure regardless of vector order. The two forms produce identical configurations. (Convention: list targets in the order their regions appear in the chart, for readability.)
- D) Only when the chart has more than two regions.

---

### 7. You write `:target [:stopped :playing]` — both atomic states inside `:playback`. What does the library do?

- A) Throws `Cannot register invalid chart` at registration time.
- B) Throws at runtime when the transition fires.
- C) Accepts the chart silently. When the transition fires, **both `:stopped` and `:playing` end up in the configuration simultaneously** inside `:playback` — a structurally invalid state. Subsequent events may produce undefined behavior. The library trusts you to put each multi-target in a different region.
- D) Picks the first target (`:stopped`) and ignores the second.

---

### 8. The chart is currently in `#{:player :playback :playing :volume :muted}`. The `:reset` transition is on the parallel with `:target [:stopped :unmuted]`. You fire `:reset`. What's the configuration after?

- A) `#{:stopped :unmuted}` — only the atomic leaves.
- B) `#{:reset}` — the chart enters a new top-level state.
- C) `#{:player :playback :stopped :volume :unmuted}` — the parallel re-activates with explicit leaves, all five elements present.
- D) `#{:player :playback :stopped :playing :volume :muted :unmuted}` — both old and new leaves coexist.

---

### 9. A multi-target `:target` references a state ID that doesn't exist (e.g., `[:stopped :unmute]`, misspelled). What happens at chart construction?

- A) The chart loads; the transition silently fails when fired.
- B) The chart loads; the runtime substitutes the closest-spelled valid state.
- C) `(t/new-testing-env ...)` throws `Cannot register invalid chart`. The chart-registration validator catches the invalid target keyword.
- D) Loads with a warning printed to stderr.

---

### 10. The exercise's pedagogical message about multi-target is that it…

- A) Is required whenever a chart has a parallel.
- B) Is required whenever a transition re-enters a parallel from outside.
- C) Lets you **precisely specify** the active leaf in each region after a transition, *overriding* the default-initial behavior. It's a *deliberate-precision tool*, not a "magic for multiple regions" tool. For this exercise's specific test, default initials happen to align with intent — so single-target works equally well.
- D) Is the only way to fire an event that affects multiple regions.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **C** | Tutorial Step 1 — multi-target enters multiple atomic states simultaneously. |
| 2 | **B** | Tutorial Step 2 + runbook Probe 1 — empirically verified. |
| 3 | **C** | Tutorial Step 2 — multi-target is for non-default initials. |
| 4 | **C** | Tutorial Step 3 — placement on the parallel makes the transition catch from any state inside it. |
| 5 | **B** | Tutorial Step 1 — internal normalization. |
| 6 | **C** | Tutorial Step 4 + runbook Probe 4 — empirically verified, order doesn't matter. |
| 7 | **C** | Tutorial Step 5 + runbook Probe 3 — empirically verified. Major footgun. |
| 8 | **C** | Tutorial Step 6 + the exit-the-whole-parallel rule from ex04. |
| 9 | **C** | The chart's configuration validator rejects unresolved target keywords at registration time. See ex02 gotcha #7 for the same error path. |
| 10 | **C** | Tutorial Step 2 — the load-bearing pedagogical takeaway. |
