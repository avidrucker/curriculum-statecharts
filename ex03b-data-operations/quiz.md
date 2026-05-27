# Quiz — ex03b: Data Model Operations & Event Data

11 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 8 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. A `script`'s `:expr` is a function. What's its expected return value?

- A) A single op map: `(ops/assign :count 0)`.
- B) A **vector** of op maps: `[(ops/assign :count 0)]`. Even when there's only one op, the vector wrapping is mandatory.
- C) A boolean — the script returns true if it succeeded.
- D) Whatever you want — the runtime ignores the return value.

---

### 2. You write `(script {:expr (fn [_ _] (ops/assign :count 0))})` (no brackets around the op). What happens when the script runs?

- A) The runtime throws `Expected vector, got map`.
- B) The runtime auto-wraps the single op in a vector for you and applies it.
- C) The runtime silently does nothing. The data model is unchanged. No error, no warning.
- D) The chart refuses to load — `statechart` validates expr return types at construction.

---

### 3. What is `on-entry` for?

- A) A one-time initialization that runs only the very first time the chart starts.
- B) A block of executable content (typically `script` elements) that runs every time a state is entered — including re-entries via self-targeting transitions or returning from a child state.
- C) An alias for `start!`.
- D) A guard that decides whether a state can be entered.

---

### 4. A targetless transition (`(transition {:event :foo} ...)` with no `:target`) does what?

- A) Errors at chart-construction time.
- B) Fires when `:foo` arrives, runs its body's side effects (e.g., scripts), but **does not change the chart's configuration**. The chart stays in whatever state declared the transition.
- C) Acts as a fallback — fires only if no other transition matches.
- D) Sends `:foo` to the parent state to handle.

---

### 5. `(handle :increment my-expr)` from `com.fulcrologic.statecharts.convenience` is equivalent to:

- A) `(state {:id :increment :expr my-expr})` — `handle` declares a state.
- B) `(transition {:event :increment} (script {:expr my-expr}))` — `handle` is sugar for a targetless transition + script. Functionally identical; just less verbose.
- C) `(on-entry {} (script {:expr my-expr}))` — `handle` is sugar for on-entry.
- D) `(guard :increment my-expr)` — `handle` declares a guard.

---

### 6. The `com.fulcrologic.statecharts.convenience` namespace's docstring says…

- A) "Use this namespace for all chart authoring — it's the primary API."
- B) "Deprecated. Migrate to the elements namespace."
- C) "ALPHA. NOT API STABLE." — the maintainer reserves the right to change names, arities, or behaviors. The underlying `transition + script + state + ...` patterns (from the elements namespace) are SCXML-aligned and stable; convenience is sugar that may evolve.
- D) Nothing — there's no warning.

---

### 7. Inside an `on-entry` script's `:expr` on the chart's *first* entry to a state, what's the value of `data`?

- A) `{}` — an empty map.
- B) `nil` — the data model hasn't been populated yet. Reading keys from it returns `nil`. The script's job is typically to *write* the initial values, not read-modify-write.
- C) Whatever was passed as the second argument to `t/new-testing-env`.
- D) The chart's `(statechart {} ...)` attributes map.

---

### 8. A test fires `(t/run-events! env {:name :set-value :data {:value 42}})`. Inside the matching handler's `:expr`, how does the expression read `42`?

- A) `(:value env)` — event payload is in env.
- B) `(:value data)` — event payload is merged into the session data.
- C) `(-> data :_event :data :value)` — the event lives at `data._event`, its payload at `data._event.data`, the application value at `:value` within the payload.
- D) `(:_event/value data)` — the event payload is auto-namespaced.

---

### 9. The exercise's `:decrement` handler is `(fn [_ data] [(ops/assign :count (max 0 (dec (:count data))))])`. What's the `(max 0 ...)` for?

- A) To prevent integer overflow on very large counts.
- B) To floor `:count` at 0 — if `:count` is already 0, `(dec 0)` is `-1`, and `(max 0 -1)` is `0`. Without it, the counter would go negative.
- C) Required by `ops/assign` — values must be passed through `max` for validation.
- D) Cosmetic — has no effect on the chart's behavior.

---

### 10. You add `:target :active` to one of the `handle`s' underlying transition. What's the practical effect?

- A) Nothing — `:target :active` is the same as no `:target` when the chart is already in `:active`.
- B) The chart will exit `:active`, then re-enter `:active`, re-firing the `on-entry` script each time. If the on-entry resets `:count` to 0, the counter will reset on every event.
- C) The chart errors — you can't target your current state.
- D) The transition becomes "external" but otherwise behaves identically.

---

### 11. `(t/data env)` returns what, before `(t/start! env)` is called?

- A) `{}` — an empty data model.
- B) An error — `t/data` requires the chart to be running.
- C) `nil`. The data model isn't populated until the chart starts and any initial on-entry scripts run.
- D) The chart's `statechart` attributes map.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 2 — scripts must return a vector of ops. |
| 2 | **C** | Empirically verified (runbook Probe 2). Silent failure — the canonical ex03b footgun. See [gotchas.md #1](./gotchas.md). |
| 3 | **B** | Tutorial Step 1 — `on-entry` fires on every state entry, not just the first time. |
| 4 | **B** | Tutorial Step 3 — targetless transitions stay in the same state. |
| 5 | **B** | Tutorial Step 4 — `handle` is sugar for a targetless transition + script. |
| 6 | **C** | The convenience namespace's own docstring. Tutorial Step 4 ("API stability warning"). |
| 7 | **B** | Empirically verified (runbook Probe in tutorial Step 1 + ex03b probe P5). On first entry, the script runs before its own ops are applied. |
| 8 | **C** | Tutorial Step 5 — event payload at `data._event.data`. |
| 9 | **B** | Floor at 0 — `(max 0 -1) = 0`. Runbook Step 4.3. |
| 10 | **B** | Self-targeting causes exit + re-entry, which re-runs `on-entry`. Runbook "Try breaking it" #3. |
| 11 | **C** | Empirically verified (runbook Probe 1). Data model is `nil` until something writes to it. |
