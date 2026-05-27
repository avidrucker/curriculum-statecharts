# Quiz — ex08: Final States, `done.state` Events, and Chart Exit

11 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 8 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. `(final {:id :done})` returns what?

- A) A regular `state` with `:final? true` flag set.
- B) A distinct element: `{:id :done, :node-type :final}`. Different `:node-type` triggers special runtime semantics (auto-fire `done.state.<parent>` on entry).
- C) A `transition` with no event.
- D) Throws — `final` requires children.

---

### 2. You write `(final {:id :done} (transition {:event :restart :target :start}))`. What happens at chart construction?

- A) The chart loads; `:restart` becomes an outgoing transition from the final.
- B) The chart loads but the transition is silently ignored.
- C) The chart throws `Illegal children of :final ...`. Finals cannot have child elements (no transitions, no on-entry, nothing). Use a regular state with no outgoing transitions if you need a "stuck" state without final semantics.
- D) The chart loads but runs at degraded performance.

---

### 3. When does the runtime auto-fire `done.state.<parent-id>`?

- A) When the chart starts.
- B) When the chart enters a `final` element whose parent's `:id` is `<parent-id>`. The event is internally generated; transitions matching it can fire as a response.
- C) When all transitions on the parent have fired.
- D) Whenever the parent state's data model is empty.

---

### 4. For a parallel state `:par` with two regions, when does `done.state.par` fire?

- A) When any one region reaches a final.
- B) When *all* regions of `:par` reach a final state. Partial completion fires the per-region `done.state.<region-id>` events but not the parallel's overall one.
- C) When the parallel is exited via an external transition.
- D) When the chart starts.

---

### 5. The exercise's chart has a top-level `(final {:id :finished})`. After `:finalize` transitions to `:finished`, what's true?

- A) The chart enters a "final mode" but keeps processing events.
- B) The configuration becomes empty (`#{}`) and `running?` becomes `false`. The chart has truly exited; subsequent events are silently dropped.
- C) The chart resets to its initial configuration.
- D) The chart raises an exception.

---

### 6. A `final` inside a compound (not top-level) — what happens when the chart enters it?

- A) The whole chart exits.
- B) The `done.state.<parent>` event fires (internal event). The chart's configuration still has the final in it, alongside its ancestors. If a transition handles the auto-event, the chart moves; if no transition handles it, the chart stays at the final (which can't transition out itself).
- C) The chart pauses until the next external event.
- D) The runtime synthesizes a `:terminate` event.

---

### 7. The auto-generated `done.state.processing` event has what shape?

- A) `{:event :done.state.processing}` — minimal.
- B) `{:type :external :name :done.state.processing :data {}}`.
- C) `{:type :internal :name :done.state.processing :sendid <final-id> :data {} ...}` — an internal event whose name follows the `done.state.<parent-id>` convention; the `:sendid` is the final's own `:id`.
- D) A keyword `:done.state.processing` (not a map).

---

### 8. You write a transition `(transition {:event :done :target :somewhere})` with the intent to handle a custom `:done` event. Why is this risky?

- A) `:done` is a reserved keyword — the chart fails to load.
- B) Via the ex02 string-prefix matching fallback, `:done` will match every `:done.state.<id>` and `:done.invoke.<id>` event the runtime synthesizes. Your transition will accidentally catch all auto-events, not just your custom one.
- C) `:done` events are queued separately and processed last.
- D) `:done` is automatically prefixed with `:user/` to avoid collisions.

---

### 9. The exercise's chart could replace `(transition {:event :done.state.processing :target :complete})` with `(transition {:event :done :target :complete})`. What's the practical difference?

- A) None — both work identically for this chart.
- B) The first form catches *only* `:done.state.processing`. The second form catches *any* event whose name starts with `:done` (including future `:done.invoke.<id>` events if you added an `invoke` element later). The first is more precise; the second is broader and risks unintended matches.
- C) The first form fails to load; only short event names are allowed.
- D) The second form is required because internal events use the dot-form pattern.

---

### 10. The chart enters `:complete` after `done.state.processing` fires. What's in the configuration at that moment?

- A) `#{:complete}` only.
- B) `#{:pipeline :complete}` — `:complete` is a child of `:pipeline`, both are in the configuration.
- C) `#{:pipeline :processing :complete}` — `:processing` is also in.
- D) The entire pre-completion configuration plus `:complete`.

---

### 11. Can you `(t/run-events! env :validation-ok)` *after* the chart has reached `:finished` (top-level final)?

- A) No — the chart throws.
- B) Yes, the chart re-enters and processes the event normally.
- C) Yes — `t/run-events!` accepts the call without error, but `running?` is `false`, the configuration is empty, no transitions can fire. The event is effectively dropped.
- D) The runtime restarts the chart from scratch.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 — `final` is a distinct element type with `:node-type :final`. |
| 2 | **C** | Empirically verified — finals can't have children. Tutorial Step 1, runbook "Try breaking it" #1. |
| 3 | **B** | Tutorial Step 2 — auto-fire on final entry. |
| 4 | **B** | Tutorial Step 3 + runbook Probe 3 — empirically verified, ALL regions must be final. |
| 5 | **B** | Tutorial Step 4 + runbook Probe 2 — empirically verified, top-level final exits the chart. |
| 6 | **B** | Tutorial Step 2 — non-top-level final fires `done.state.<parent>` but doesn't exit the chart. |
| 7 | **C** | Tutorial Step 2 + runbook Probe 4 — empirically verified event shape. |
| 8 | **B** | Tutorial Step 6 — the cross-module string-prefix matching collision. |
| 9 | **B** | Same as Q8 — precision vs. broad matching. |
| 10 | **B** | Compound state semantics from ex02 — `:pipeline` stays in the configuration alongside its active child `:complete`. |
| 11 | **C** | Tutorial Step 4 — after chart exit, events are silently dropped. The library doesn't error on the call. |
