# Quiz — ex01: Compound States & Configuration

8 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 6 right, re-read the section of the tutorial named in the "Quick reason" column for that question, then re-take.

---

### 1. In the `com.fulcrologic/statecharts` library, what *is* a statechart?

- A) A class instance whose internal fields are mutated as events arrive.
- B) A plain Clojure data structure (a map) produced by the `statechart` factory function. The runtime — a separate "env" — actually executes it.
- C) A long-running thread that receives events via a `core.async` channel.
- D) A reagent atom that re-renders subscribers when its state changes.

---

### 2. The traffic-light chart in ex01 declares three states (`:red`, `:green`, `:yellow`) under the chart root, with no explicit `:initial`. Which state does the chart enter when `(t/start! env)` runs?

- A) The state whose `:id` is alphabetically first — `:green`.
- B) A pseudo-random choice among the three top-level states.
- C) The state declared first in **document order** — `:red` — because document order is the fallback when no `:initial` is given.
- D) An error — `start!` requires an explicit `:initial` on every compound state, including the chart root.

---

### 3. Where is a transition declared in the chart definition?

- A) At the chart root, in a `:transitions` vector that maps `[from-state event]` to `target-state`.
- B) Inside the **source** state element — the state the transition *starts from*. The enclosing state is the "from"; the transition only names the "to" via `:target`.
- C) Inside the **target** state element — the state the transition lands in.
- D) Anywhere — the runtime scans the whole chart for `transition` elements and routes them based on `:event` alone.

---

### 4. The chart is in `:green`. You call `(t/in? env :green)`. What does it return, and why?

- A) `false` unless you also pass the chart's data model as a third argument.
- B) An error — `t/in?` requires the chart to be stopped before you can introspect it.
- C) `true`, because `t/in?` is a `contains?` check against the chart's configuration set — and `:green` (the currently active atomic state) is a member of that set.
- D) `true` only if `:green` is the *deepest* active state AND you also pass `{:strict-leaf? true}` as an option.

---

### 5. The chart is in `:red`. You fire `(t/run-events! env :bogus)`, where `:bogus` doesn't match any transition. What happens?

- A) The chart throws an `ExceptionInfo` because no transition handles the event.
- B) The chart resets to its initial configuration.
- C) The event is queued and re-fired automatically the next time `run-events!` is called.
- D) The event is consumed and dropped; the chart's configuration is unchanged. Unknown events don't error — they're ignored.

---

### 6. Why doesn't ex01 need a `(state {:id :traffic-light} …)` element wrapping `:red`, `:green`, and `:yellow`?

- A) Because none of the three states have children of their own — wrapper states only exist when there's nesting to do.
- B) Because the `statechart` factory auto-generates a root node (`:id :ROOT`, `:node-type :statechart`) that serves as the parent of all top-level states. Adding an explicit wrapper would be redundant.
- C) Because the library forbids more than one level of nesting in any chart.
- D) Because explicit compound states are reserved for use with the `parallel` element (ex04).

---

### 7. A transition is written `(transition {:event :next :target :green})`. What does `:target` reference?

- A) A function called when the transition fires; it receives the env and returns the next state's `:id`.
- B) A free-form text label shown in a visual debugger; the runtime ignores it.
- C) The `:id` of an existing state in the chart. If no state has that `:id`, the runtime errors when the transition fires (or possibly at chart construction time, depending on validation).
- D) The Clojure `def` name of a state — i.e., it would be the symbol `green`, not the keyword `:green`.

---

### 8. What's the relationship between the `traffic-light` chart definition and the value returned by `(t/new-testing-env {:statechart traffic-light …})`?

- A) They're the same thing — `new-testing-env` is a no-op wrapper around the chart.
- B) The chart is the *definition* (data). The env is the *runtime container* — it holds the chart definition plus the current configuration, the event queue, and the data model. Multiple envs can wrap the same chart.
- C) The env is generated reflectively from the chart's `:id` — it shares mutable state with the chart definition by reference.
- D) The chart subscribes to the env; firing an event on the env notifies all subscribed charts.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | A chart is plain data — see tutorial Step 1 ("A statechart is data"). The runtime is a separate concept. |
| 2 | **C** | Document order is the fallback when no `:initial` is given — tutorial Step 4 ("The initial state is the first child in document order"). |
| 3 | **B** | Transitions live inside the source state — tutorial Step 3 ("Transitions live inside states"). The enclosing state is the "from." |
| 4 | **C** | `t/in?` is `(contains? configuration state-name)` — a set-membership check. See tutorial Step 5 and the `configuration check via t/in?` entry in the per-exercise glossary. |
| 5 | **D** | Unknown events are dropped, not errors — tutorial Step 6's last paragraph; reinforced in runbook Probe 3. |
| 6 | **B** | The `statechart` factory auto-generates the root with `:id :ROOT` — tutorial Step 2 ("States are nodes in a tree") and the `chart root (:ROOT)` glossary entry. |
| 7 | **C** | `:target` names a state by its `:id`. The runtime resolves the ID to the state element. See tutorial Step 3. |
| 8 | **B** | The chart is data; the env is the runtime — tutorial Step 1's distinction. Reinforced in runbook Probe 2 (configuration changes after `start!` but the chart def doesn't). |
