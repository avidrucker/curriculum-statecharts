# Quiz — ex09: Child Chart Invocations

13 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 10 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. The `invoke` element is declared inside a state. What does it do when the chart enters that state?

- A) Adds the named child to the parent's configuration.
- B) Starts a separate session running the child chart referenced by `:src` (looked up in the chart registry). The child has its own session-id (= the invoke's `:id`), its own working memory, its own configuration.
- C) Synchronously runs the child chart to completion before returning.
- D) Imports the child's states into the parent's chart.

---

### 2. The invoke's `:id` becomes…

- A) The state's `:id` (overriding any explicit declaration).
- B) The child session's session-id, AND the suffix of the auto-event name `:done.invoke.<id>`.
- C) A label for the parent's logging.
- D) A type discriminator (must match `:type`).

---

### 3. The invoke's `:src` references…

- A) A namespace where the child chart's source lives.
- B) A URL to fetch the chart from over HTTP.
- C) A registry key. Before invocation works, both the parent and child must be registered via `simple/register!`.
- D) The path to a `.cljc` file on disk.

---

### 4. After the parent fires `:pay` (entering `:processing-payment`), what's `(::sc/configuration (sp/get-working-memory wmstore env :payment-process))`?

- A) `nil` — `:payment-process` is the invokeid, not a session-id.
- B) `#{:charging}` — the child session is now running, in its initial state.
- C) `#{:processing-payment :charging}` — the parent's invoking state plus the child's state, merged.
- D) `#{:payment :charging}` — the parent's invoke node's id plus child state.

---

### 5. When does `done.invoke.<invokeid>` fire?

- A) When the parent's state containing the invoke is exited.
- B) When the child reaches a final state (any final inside the child chart, not just the chart-root final).
- C) When the child chart's working memory has running? = false — i.e., it has reached a top-level final.
- D) When the parent fires a `:done` event manually.

Hmm — which final triggers the auto-event?

---

### 6. (Continuing from Q5) For the exercise's `payment-chart`, which has `:charging → :charged` where `:charged` is a final at the chart's top level — when does the auto-event fire?

- A) When the child enters `:charging` (which already happens at child start).
- B) When the child's `:charge-complete` event fires and the child enters `:charged` (a top-level final → child exits and the runtime fires the auto-event back to the parent).
- C) When the parent fires `:charge-complete`.
- D) When the test calls `sp/receive-events!`.

---

### 7. Why does the exercise use `simple/simple-env` instead of `t/new-testing-env`?

- A) `t/new-testing-env` doesn't support compound charts.
- B) `t/new-testing-env` uses `MockInvocations` — a stub that doesn't actually start child charts. For invocation-bearing charts to actually execute, the test needs the real invocation processor, which `simple/simple-env` provides.
- C) `simple/simple-env` is faster.
- D) `t/new-testing-env` requires Fulcro to be installed.

---

### 8. Inside the test, you call `(sp/get-working-memory wmstore env :payment-process)` to access the child's state. Why use `:payment-process` and not the chart's name?

- A) Because the chart name (`:payment-chart`) is the registry key, not the session-id. The invokeid `:payment-process` is the session-id by which the working-memory-store identifies the child.
- B) The library uses random session-ids; `:payment-process` is a guess.
- C) The child has multiple session-ids; `:payment-process` is one of them.
- D) `:payment-chart` would work too — they're aliases.

---

### 9. What happens to the child session if the parent transitions out of `:processing-payment` *before* the child completes (e.g., via a `:cancel` transition)?

- A) The child continues running in the background until it reaches its own final.
- B) The child is **automatically terminated** when the parent exits the invoking state. The child's working memory is removed; the child can't fire events back to the parent.
- C) The runtime queues a `:cancel` event for the child to handle.
- D) An error is raised.

---

### 10. The test's `sp/receive-events!` call has a custom handler function. What does that function do?

- A) Filters out events that aren't `done.invoke`.
- B) Routes each event from the event queue to its `:target` session by looking up that session's working memory and processing the event there. This is the manual replacement for `t/run-events!`'s built-in routing.
- C) Logs events for debugging.
- D) Sends each event to *all* registered sessions.

---

### 11. A transition listening for `:done.invoke.payment-process`. If you accidentally write `:done.invoke.payment` (missing `-process`), what happens?

- A) The chart fails to register.
- B) The transition matches `:done.invoke.payment-process` because of ex02's string-prefix matching fallback (`:done.invoke.payment` is a string-prefix of `:done.invoke.payment-process`). The test may pass *by accident*.
- C) The transition matches `:done.invoke.payment-process` because all `done.invoke.*` events match all `done.invoke.*` transitions.
- D) The chart loads but the transition never fires; the test fails.

---

### 12. Inside the `done.invoke.payment-process` event map, which key tells you which session it should be routed to?

- A) `:invokeid` — that's the parent's session-id.
- B) `:target` — set to the parent's session-id by the runtime when the event was synthesized. The test's `sp/receive-events!` handler reads `(:target event)` and routes accordingly.
- C) `:com.fulcrologic.statecharts/source-session-id` — that's the parent.
- D) `:name` — the event name encodes the routing.

---

### 13. After the test completes successfully, the parent is in `:confirmation`. What's `(some? (sp/get-working-memory wmstore env :payment-process))` (the child's working memory)?

- A) `true` — the child reached its final but its working memory is preserved for history.
- B) `false` — when the parent left `:processing-payment` (to enter `:confirmation`), the invoke was implicitly cancelled. The child session no longer exists in the working-memory-store.
- C) `true` initially, then `false` after `sp/receive-events!` returns.
- D) Depends on whether the child's final was top-level or nested.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 — invoke starts a separate session. |
| 2 | **B** | Tutorial Step 1 + glossary `invoke` — `:id` is both session-id and event-name suffix. |
| 3 | **C** | Tutorial Step 1 — `:src` is a registry key. |
| 4 | **B** | Tutorial Step 2 + runbook Probe 1 — empirically verified child config. |
| 5 | **C** | Tutorial Step 3 — `done.invoke.<invokeid>` fires when the *child chart* reaches a top-level final (which means the child has fully exited). |
| 6 | **B** | The `:charged` final is at the chart's root → child exits → auto-event fires. |
| 7 | **B** | Tutorial Step 5 + runbook Probe 3 — empirically verified that t/new-testing-env mocks invocations. |
| 8 | **A** | Tutorial Step 2 — invokeid (= `:id` from the invoke) is the session-id. |
| 9 | **B** | Tutorial Step 4 + runbook Probe 4 — empirically verified auto-cancellation. |
| 10 | **B** | Tutorial Step 6 — manual orchestration replaces `t/run-events!`. |
| 11 | **B** | ex02's string-prefix matching fallback bites again. See ex02 gotcha #1 + ex08 gotcha #2. |
| 12 | **B** | The `:target` field is set by the runtime when synthesizing the event. Verified in runbook Probe 2. |
| 13 | **B** | Tutorial Step 4 — exiting the invoking state auto-cancels the child. Even though the child had already finalized, leaving the invoking state cleans up its working memory. |
