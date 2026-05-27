# Quiz — ex03c: Convenience Helpers (`on`, `handle`, `choice`)

11 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 8 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. `(on :submit :checking)` is equivalent to which of the following?

- A) `(state {:id :submit} (on-entry {} (script {:expr ...})))` — `on` declares a state with an entry action.
- B) `(transition {:event :submit :target :checking})` — pure sugar; same chart element under a different surface syntax.
- C) `(handle :submit (fn [_ _] [(ops/assign :state :checking)]))` — `on` is sugar around `handle`.
- D) Nothing — `on` is reserved and can't be replaced by another form.

---

### 2. What does `(choice {:id :auth-check} pred? :a :else :b)` evaluate to, structurally?

- A) A new chart-element type (`:node-type :choice`) that the runtime treats specially.
- B) A state element (`:node-type :state`) with `:diagram/prototype :choice` and two eventless transitions as children — one with `:cond pred?` and `:target :a`, one with `:target :b` (no `:event`, no `:cond`).
- C) A pair of `cond` forms wrapped in `do`.
- D) A `parallel` element containing two regions.

---

### 3. An "eventless transition" is one that has…

- A) No `:event` key (or `:event nil`). It fires automatically when its state is entered (or during chart settling), gated only by its `:cond` if present.
- B) An `:event :*` wildcard that matches any event.
- C) No `:target` — it's another name for a targetless transition.
- D) An `:event` key set to a symbol rather than a keyword.

---

### 4. After `(t/run-events! env :submit)` on the solution chart (where password is `"secret"`), what does `(t/in? env :checking)` return?

- A) `true` — the chart is in `:checking` for the rest of the test.
- B) `false`. The chart entered `:checking`, then its initial child (the `:auth-check` choice) immediately dispatched via an eventless transition to `:authenticated`. `:checking` and `:auth-check` are both **transient states** — the chart passes through, never lingers.
- C) An error — `t/in?` can't be called on compound states.
- D) `true` *or* `false` non-deterministically.

---

### 5. The choice in the exercise has an `:else :rejected` clause. What would happen if you removed `:else` and the password were `"wrong"`?

- A) The chart would silently snap back to `:idle`.
- B) The chart would land in `:authenticated` as a fail-safe default.
- C) An exception would be raised at chart-construction time.
- D) The chart would get **stuck** in `:auth-check` (the choice state). No transition is eligible (the only conditional predicate returns false; there's no fallback). `(t/in? env :auth-check)` would return `true` indefinitely.

---

### 6. Inside a choice's predicate that was reached via `(on :go :decide-state-containing-choice)`, what's the value of `(-> data :_event)`?

- A) `nil` — eventless transitions never see an event.
- B) The map for the `:go` event that led to the choice, including `:name :go`, `:data {...}`, etc. The `:_event` from the triggering event carries into the choice's predicates.
- C) The map for the choice's *own* "synthetic" entry event (named `:enter`).
- D) Always `{:name :system/eventless}` — a placeholder for eventless triggers.

---

### 7. The `com.fulcrologic.statecharts.convenience` namespace's docstring says…

- A) "Stable, recommended for production charts."
- B) "**ALPHA. NOT API STABLE.**" The maintainer reserves the right to change names, arities, or behaviors. The underlying elements-namespace forms are the stability anchor.
- C) "Deprecated. Use the elements namespace instead."
- D) Nothing — there's no warning.

---

### 8. You have a chart that uses both `(on :foo :bar)` and `(transition {:event :foo :target :bar})` at different points. The runtime treats them…

- A) Differently — `on` transitions are higher-priority than verbose ones.
- B) Differently — `on` transitions are skipped if any other transition matches the event.
- C) Identically. Both emit the same `:node-type :transition` element shape; the runtime doesn't know (or care) which surface form was used.
- D) Differently — only one form can appear per state.

---

### 9. The exercise's `:idle` state uses `handle` to store the password. Why not put the password-storage logic inside the `:submit` transition?

- A) Because `:submit` transitions must use `on`, which can't carry a script.
- B) Pedagogical / architectural reasons. The chart's design separates "data acquisition" (`:enter-password`) from "decision" (`:submit` → `:checking` → choice). Both could be combined, but two events makes the user-facing flow clearer (`enter, then submit`) and lets the chart handle pre-submit password updates without leaving `:idle`.
- C) Because `:submit` arrives before `:enter-password` deterministically.
- D) Because `handle` and `on` are mutually exclusive within a single state.

---

### 10. Inside a `choice` predicate that was entered automatically (not via an event), what is `(-> data :_event)`?

- A) The system-event `:initialize`.
- B) `nil`. With no triggering event, there's no `:_event` to inject. The predicate runs with `data` containing only the session-data keys, plus `:_event` valued `nil` (or absent — implementation detail).
- C) The chart-options map.
- D) The same as for event-triggered entries — there's no distinction.

---

### 11. The convenience namespace is "ALPHA. NOT API STABLE." Given that, the curriculum's recommendation is…

- A) Avoid the convenience namespace entirely.
- B) Use convenience for everything — its sugar is worth the future risk.
- C) Use convenience for curriculum / exploration / quick prototypes where stability isn't paramount; prefer the elements-namespace forms for long-lived production charts where library upgrades shouldn't break your code. The two forms can be mixed in the same chart.
- D) Use whichever form is alphabetically first in each context.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 2 — `on` is pure sugar for a basic transition. |
| 2 | **B** | Tutorial Step 4 + runbook Probe 2 — choice is a state with `:diagram/prototype :choice` and eventless transition children. |
| 3 | **A** | Tutorial Step 5 — no `:event` key. Fires on state entry / during settling. |
| 4 | **B** | Tutorial Step 4 + runbook Probe 4 — choice states are transient; the chart passes through. |
| 5 | **D** | Tutorial Step 7 + gotchas #1 + runbook "Try breaking it" #1 — empirically verified stuck-in-choice failure mode. |
| 6 | **B** | Tutorial Step 5 + runbook Probe 3 — empirically verified. |
| 7 | **B** | The namespace's own docstring; tutorial Step 1. |
| 8 | **C** | Tutorial Step 1 — convenience is pure sugar; runtime sees identical elements. |
| 9 | **B** | Tutorial Step 6 + a glance at the test ordering. Multiple-event flow is a design choice for clarity, not a library requirement. |
| 10 | **B** | Tutorial Step 5 + runbook Probe 3 (second case — auto-entry) — empirically verified, `:_event` is nil. |
| 11 | **C** | Tutorial Step 1 — mix-and-match deliberately based on the chart's longevity. |
