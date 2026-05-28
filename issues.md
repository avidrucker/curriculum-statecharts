# Issues — curriculum-wide tracker

Curriculum-level items that are **plausible but not yet empirically verified**, or **known imperfections** awaiting a follow-up pass. Distinct from:

- **[`ask_tony.md`](./ask_tony.md)** — questions about the library author's *intent* (we know the behavior, but want to know the contract).
- **[`ex03-guards/issues.md`](./ex03-guards/issues.md)** — per-module deferred items specific to ex03.

Format per entry: short title, what was claimed, where it was claimed, why it's not (fully) verified, how to investigate.

When you resolve an item, move it to the "Resolved" section with a one-line outcome + the file you updated.

---

## Open — verification queue

### 1. Does `history` inside an invoked child chart work as expected?

**Where claimed.** [`ex09-invocations/exec_summary.md`](./ex09-invocations/exec_summary.md) "Composes with" section: "each child session is a separate chart with its own working memory, so it has its own history bookkeeping."

**Why not verified.** I never explicitly probed this during ex09 authoring or the review pass. The claim is plausible from the library's design (each invocation gets its own session, separate working memory, separate history bookkeeping), but plausibility ≠ verification.

**How to investigate.** Construct a parent chart that invokes a child. The child should have a compound state containing a history node. Drive the child into a deep state. Exit the child (drive it to its final). Re-invoke (e.g., make the parent re-enter the invoking state). Verify the child's history was preserved (or empirically observe whether it was).

**Impact.** Low-medium. Real Fulcro production charts may rely on this combination; if it doesn't work as expected, the curriculum's claim is misleading. If it does work, the claim is correct but currently un-cited.

---

### 2. `:autoforward true` behavior nuances on `invoke`

**Where claimed.** [`ex09-invocations/exec_summary.md`](./ex09-invocations/exec_summary.md) "Gotchas to remember": "`:autoforward true` is rarely what you want. Forwards every parent event to the child — useful for 'child wraps parent' semantics, surprising in most other cases." Also in [`ex09-invocations/gotchas.md`](./ex09-invocations/gotchas.md) #10.

**Why not verified.** The claim about behavior matches the library's docstring on `invoke`, but I never empirically tested `:autoforward true` to confirm which events get forwarded, whether the auto-events (`done.invoke.<id>`) are forwarded, and what happens if both parent and child have transitions for the same event name.

**How to investigate.** Probe `:autoforward true` with a parent + child that share an event name. Fire the event on the parent. Verify whether the child also processed it.

**Impact.** Low. The curriculum doesn't lean on `:autoforward`; the recommendation is to leave it off. But the gotcha framing could be sharpened with empirical detail.

---

### 3. `done-data` element behavior on `final`

**Where claimed.** [`ex08-final-and-done/gotchas.md`](./ex08-final-and-done/gotchas.md) #8 and [`ex08-final-and-done/exec_summary.md`](./ex08-final-and-done/exec_summary.md): "`done-data` (optional element) can attach data to the auto-event. Not used in this exercise; useful for parallel-completing aggregations."

**Why not verified.** The library source has a `done-data` element, but I didn't probe its actual behavior — specifically, does the data show up in the `:data` field of the auto-event, or somewhere else? Does it execute expressions, or just attach static values?

**How to investigate.** Construct a chart with `(final {:id :x} (done-data {} <expr>))`. Drive the chart to enter `:x`. Capture the auto-event from the event queue. Inspect `(:data event)` and other fields for the done-data's value.

**Impact.** Low. The curriculum doesn't use `done-data` in any exercise; the mention is for awareness only. But the gotcha's framing could be tightened with empirical confirmation.

---

### 4. Variable name `cart-chart` in ex03b is an upstream artifact

**Where.** [`statechart-exercises`](../statechart-exercises)`/src/exercises/ex03b_data_operations.cljc` and the corresponding solution file use `(def cart-chart ...)` but the topic is a counter, not a cart.

**Why this matters.** The curriculum's [`ex03b-data-operations/exec_summary.md`](./ex03b-data-operations/exec_summary.md) and runbook both note the naming inconsistency. The right fix is to rename in the upstream `statechart-exercises` repo and update the curriculum's references. The wrong fix is to live with the discrepancy indefinitely.

**How to investigate.** Open a PR upstream renaming `cart-chart` → `counter-chart` (or similar). After upstream merges, update the curriculum's references.

**Impact.** Cosmetic. Doesn't affect curriculum correctness; affects readability.

---

### 5. Footnote pattern `[^p#]` retrofit for ex01–ex05

**Where.** Modules ex06–ex09 use the `[^p#]` footnote marker pattern to trace empirical claims to specific probes (verified empirically section at the end of each doc). Modules ex01–ex05 don't (the pattern was adopted partway through authoring).

**Why this matters.** Inconsistency. A reader scanning for "what's been verified" sees ex06–ex09 explicitly tagged but has to infer the same level of empirical backing for ex01–ex05 from context (publication-order + review-pass history).

**How to investigate.** Retroactively add `[^p#]` markers to ex01–ex05 tutorials, runbooks, gotchas. Each major empirical claim gets a marker; bottom of each file gets a "Verified empirically" block listing the probes.

**Impact.** Medium for thoroughness; low for actual correctness (the claims are already verified from earlier review passes — what's missing is the explicit traceability).

**Cost.** Significant — 25 files (5 modules × 5 docs each — though only the empirical-claim files need updates, so probably ~15 files). Was considered + deferred during the v1 review pass.

---

### 6. Meta-statechart diagram (concepts-map Diagram 7) blends procedure phases with runtime states

**Where.** [`concepts-map.md`](./concepts-map.md) Diagram 7 ("A statechart of the statechart lifecycle").

**Concern.** The diagram uses `stateDiagram-v2` syntax to represent the chart's lifecycle, but the "states" are a mix of procedural phases (Constructed, Registered) and actual runtime states (Running, Exited). The Settling sub-state captures the eventless cascade accurately but the construction/registration steps aren't really states the runtime models.

**Why this matters.** A reader new to statecharts might think `Constructed` and `Registered` are real chart-states they can be "in," whereas they're really phases of the authoring/setup workflow. The diagram is labeled "tongue-in-cheek but instructive" but the warning could be sharper.

**How to investigate.** Decide whether to:

- (a) Reorder/relabel the diagram to make the procedural-vs-runtime distinction explicit (e.g., split into two diagrams).
- (b) Add a more prominent warning above the diagram.
- (c) Leave as-is on the grounds that the existing framing is sufficient.

**Impact.** Low-medium. The diagram is in the "advanced/complete" tier of the concepts-map, so readers reaching it should be sophisticated enough to handle the blur. But a sharper framing wouldn't hurt.

---

## Resolved

*(none yet)*
