# Quiz — ex04: Parallel Regions

13 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 10 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. What's the structural difference between `(state {:id :x} childA childB)` and `(parallel {:id :x} childA childB)`?

- A) Cosmetic only. The runtime treats them identically.
- B) The `state` form makes exactly one of `childA` or `childB` active at a time (whichever is first in document order, or the one named by `:initial`). The `parallel` form makes **both** active simultaneously.
- C) `parallel` is just `state` with `:initial` set to a special "all" value.
- D) `parallel` allows children to be transitioned to from outside; `state` doesn't.

---

### 2. Immediately after `(t/start! env)` runs the media-player chart, the configuration set contains…

- A) `#{:stopped}` only — `t/in?` reports the deepest active state.
- B) `#{:player :stopped :unmuted}` — the parallel and both atomic leaves.
- C) `#{:player :playback :stopped :volume :unmuted}` — the parallel, both regions, and both atomic leaves. (`:ROOT` is excluded as always.)
- D) `#{:ROOT :player :playback :stopped :volume :unmuted}` — same as C but including `:ROOT`.

---

### 3. The chart is in `:playing` (in `:playback`) and `:unmuted` (in `:volume`). You fire `:play`. What happens?

- A) Nothing — `:play` is not a valid event for the chart's current configuration.
- B) The chart errors because `:play` has been handled.
- C) `:play` is broadcast to both regions. `:playback`'s active state (`:playing`) has no transition on `:play`, so it stays. `:volume`'s active state (`:unmuted`) has no transition on `:play`, so it stays. The chart is unchanged.
- D) The chart resets to `:stopped` and `:unmuted`.

---

### 4. The chart is in `:stopped` and `:unmuted`. You fire `:toggle-mute`. What's the new configuration?

- A) `#{:player :playback :playing :volume :muted}` — both regions update.
- B) `#{:player :playback :stopped :volume :muted}` — only `:volume` updates; `:playback` is unaffected.
- C) `#{:muted}` — `:toggle-mute` exits the whole parallel.
- D) `#{:player :playback :stopped :volume :unmuted}` — same as before; `:toggle-mute` doesn't match any transition.

---

### 5. The exercise's `:volume` region declares no transition for `:play`. When `:play` fires while `:volume` is in `:unmuted`, what does the runtime do with the event *in `:volume`*?

- A) Throws an error — every region must handle every event.
- B) Silently drops the event for `:volume`. The region stays in `:unmuted`. Meanwhile, `:playback`'s transitions on `:play` *do* fire.
- C) Forwards `:play` to `:playback` automatically.
- D) Routes `:play` to a `:default` handler at the chart root.

---

### 6. A `(parallel ...)` element is declared as the chart's only top-level child. If you replace it with `(state ...)` (keeping the same children), what observably changes?

- A) Nothing — the runtime treats `state` and `parallel` identically at the top level.
- B) The chart now has **exactly one** of `:playback` or `:volume` active at a time (whichever is first in document order — `:playback`). `:volume` and `:unmuted` are never active. `(t/in? env :unmuted)` returns `false` after `start!`. This is the canonical ex04 footgun (see [gotchas.md #1](./gotchas.md)).
- C) The chart fails to load with `Cannot register invalid chart`.
- D) The chart loads but runs slowly because the runtime can't decide which child to activate.

---

### 7. A transition declared inside `:playing` targets `:off`, where `:off` is a state declared at the chart root (outside the `:player` parallel). What happens when this transition fires?

- A) Only `:playing` exits; the rest of the chart stays put (including the `:volume` region).
- B) The transition errors at runtime — you can't escape a parallel from one of its children.
- C) The whole `:player` parallel exits — both regions, the parallel itself — and the chart enters `:off`. The configuration after the transition is `#{:off}`. This is SCXML's LCA (Least Common Ancestor) rule for transition exit sets.
- D) Only the `:playback` region exits; `:volume` continues independently.

---

### 8. A transition declared **on the `parallel` element itself** (as a direct child of `parallel`, alongside the regions) listens for `:hard-reset` and targets a state outside the parallel. When does it fire?

- A) Never — transitions can't be declared on a `parallel` element.
- B) Only when both regions are simultaneously in a specific configuration.
- C) Whenever the chart is "in" the parallel (which is always, while it's active) and `:hard-reset` arrives, regardless of which atomic leaves each region currently holds. Acts like a "machine-wide" event handler.
- D) Only on chart startup.

---

### 9. The configuration set has multiple atomic leaves active. `(t/in? env :stopped)` returns `true`. What does that mean?

- A) `:stopped` is the chart's only active state.
- B) The chart has a special "multi-state" mode set.
- C) `:stopped` is a member of the configuration set. The runtime calls `(contains? configuration-set :stopped)` and gets true. **Same `t/in?` semantics as every prior exercise** — the only difference is the set has more members.
- D) `:stopped` is the deepest active state in document order.

---

### 10. You declare a `:reset` transition in `:playing` that targets `:stopped` (within `:playback`). When `:reset` fires, the `:volume` region is…

- A) Reset to `:unmuted` (parallels are coupled).
- B) Exited entirely (the parallel collapses on any region's transition).
- C) Unchanged. The transition targets a state *within* `:playback`, so the LCA is `:playback`, and only `:playback` re-enters. `:volume` continues in whatever state it had.
- D) Re-entered at its initial state (transitions trigger full parallel re-entry).

---

### 11. Each child of a `parallel` element is called a…

- A) sub-state.
- B) atomic region.
- C) **region** — a (typically compound) state that operates independently from its sibling regions inside the parallel.
- D) leaf.

---

### 12. The runbook's Step 4.1 declares the skeleton chart with just two atomic-state-per-region. Both `(t/in? env :stopped)` and `(t/in? env :unmuted)` pass. Why?

- A) The chart got lucky — atomic states with no body default to "active."
- B) The `parallel` element activates both regions on entry. Each region enters its first-document-order child. `:stopped` and `:unmuted` are the only atomic states declared in their respective regions, so they're the active leaves. Both are in the configuration; `t/in?` is set-membership.
- C) `t/in?` returns true for any declared state, whether or not it's active.
- D) `start!` automatically populates the configuration with all top-level atomic states.

---

### 13. The exercise's chart is independent across regions (muting doesn't affect playback). What makes this independence work?

- A) A `:independence` flag on the `parallel` element.
- B) The library detects independent concerns automatically.
- C) Structural: each region's states declare transitions only for events that region cares about. The `:volume` region's states have transitions only for `:toggle-mute`; the `:playback` region's states have transitions only for `:play`, `:pause`, `:stop`. The chart doesn't know "these are independent" — independence is the natural result of which transitions are declared where.
- D) Regions run on separate threads and can't see each other's events.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 — the load-bearing semantic difference. |
| 2 | **C** | Tutorial Step 2 + runbook Probe 1 — five-element configuration, `:ROOT` excluded. |
| 3 | **C** | Tutorial Step 4 — event broadcast; events with no matching transition in any region are silently dropped per region. |
| 4 | **B** | Tutorial Step 5 — only `:volume` declares `:toggle-mute`; `:playback` is unaffected (independence is structural). |
| 5 | **B** | Tutorial Step 4 — events that no region handles are silently dropped *for that region*. |
| 6 | **B** | Tutorial Step 1 + runbook "Try breaking it" #1 — the canonical footgun. |
| 7 | **C** | Tutorial Step 6 — the exit-the-whole-parallel rule, empirically verified in runbook Probe 5. |
| 8 | **C** | Tutorial Step 7 + ex04 probe 7 — empirically verified. |
| 9 | **C** | Tutorial Step 3 — `t/in?` is `contains?`; the set just has more members. |
| 10 | **C** | Tutorial Step 6 (LCA rule, in reverse) — transitions targeting within a region keep other regions intact. |
| 11 | **C** | Tutorial Step 2 — "region" is the standard SCXML term for parallel children. |
| 12 | **B** | Runbook Step 4.1 — parallel activates all regions; each goes to its first-document-order child. |
| 13 | **C** | Tutorial Step 5 — independence is structural, not declared. |
