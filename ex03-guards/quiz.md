# Quiz — ex03: Guards (Conditions) & Document Order

12 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 9 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. What's the function signature of a `:cond` guard?

- A) `(fn [event data] boolean)` — event first.
- B) `(fn [env data] boolean)` — env first, data second. The runtime calls the guard with two positional args, in that order.
- C) `(fn [data env event] boolean)` — three args.
- D) `(fn [data] boolean)` — single arg.

---

### 2. Two transitions on the same state both match `:submit`. The first has a `:cond` that returns `false`. The second has no `:cond` at all. What happens?

- A) An error — when multiple transitions match, the runtime must reject the chart at construction time.
- B) Neither fires — without an explicit "default" marker, the runtime drops the event.
- C) The second transition fires. A missing `:cond` is treated as unconditionally true, and the runtime picks the first transition in document order whose guard passes (here, the second one).
- D) Both fire — the runtime runs every matching transition in sequence.

---

### 3. A guard returns the string `"false"`. The transition has `:event :submit`. The event arrives. Does the transition fire?

- A) No — `"false"` parses as a falsy boolean.
- B) Yes — `"false"` is a string, and any non-nil non-`false` value in Clojure is truthy. The transition fires.
- C) Error — guards must return a boolean strictly.
- D) Depends on the chart options.

---

### 4. The chart has two `:submit` transitions in `:login`, both guarded. The first guard returns `true` and the chart fires. Is the second guard called?

- A) Yes — the runtime evaluates all guards on matching transitions before deciding which to fire.
- B) Yes, but only its return value is discarded.
- C) No. The runtime *short-circuits* — once a matching transition's guard passes, it fires and subsequent transitions are not consulted.
- D) Sometimes, depending on the `::sc/document-order` chart option.

---

### 5. Inside a guard, `data` is `{:password "secret", :_event {:name :submit, :data {:foo 42}}}`. Which expression reads the password the user typed *with the event*?

- A) `(:password data)` — top-level keys are always the event payload.
- B) `(get-in data [:_event :data :password])` — the event payload lives at `:_event :data`.
- C) `(:event-password data)` — there's an auto-injected convenience key.
- D) `(get env :event :password)` — event data is in `env`, not `data`.

---

### 6. `(op/assign :password "secret")` is evaluated at the REPL. What does it *return*, and does it have any *side effect*?

- A) Returns `:ok`; side effect: writes `"secret"` to the in-process data model.
- B) Returns `nil`; side effect: queues an update on the next event tick.
- C) Returns a map like `{:op :assign :data {:password "secret"}}`; no side effect. It's an operation *descriptor* that has to be fed into a sink (e.g., `goto-configuration!` or an `on-entry` action) for the mutation to happen.
- D) Throws — `op/assign` must be called from inside a transition body.

---

### 7. What's the purpose of `t/goto-configuration!`?

- A) A production API for jumping the chart to an arbitrary configuration from outside the chart's machinery.
- B) A test-time-only helper that sets the data model (via the ops you pass) and the configuration set (to the leaf states you name), bypassing the normal entry actions. Useful for setting up scenarios that would otherwise require driving the chart through many events.
- C) A debugging tool that prints the current configuration to stdout.
- D) An alias for `t/start!`.

---

### 8. With the exercise's chart, after `(t/goto-configuration! env [(op/assign :password "secret")] #{:login})`, what's the configuration set?

- A) `#{:login}` — same as the result of a normal `start!`.
- B) `#{:login :ROOT}` — `goto-configuration!` uses `configuration-for-states`, which includes ancestors *and* the chart root. (Note: this differs from the normal `start!` lifecycle, which excludes `:ROOT`.)
- C) `#{}` — empty until an event is fired.
- D) `#{:login :authenticated}` — the future possibility states are also included.

---

### 9. A guard throws an `ex-info`. What does `(t/run-events! env :submit)` do?

- A) Re-throws the exception immediately; the env is in an undefined state.
- B) Stops processing and leaves the chart in its prior state.
- C) Treats the throwing guard as a failed guard (returning false). Logs the exception at DEBUG level. Falls through to the next matching transition.
- D) Halts the chart permanently — the env can no longer process events.

---

### 10. You have two transitions, both guarded:

```clojure
(transition {:event :go :cond expensive-check :target :a})
(transition {:event :go :cond cheap-check     :target :b})
```

You want to optimize. Which transition should you declare first?

- A) Doesn't matter — the runtime evaluates both guards in parallel.
- B) Put `cheap-check` first. The runtime short-circuits on the first passing guard; if the cheap one passes often, you'll skip the expensive one frequently. (Caveat: only swap order when the *semantics* still work — order changes which transition fires when both would pass.)
- C) Put `expensive-check` first. The runtime caches its result.
- D) The runtime auto-reorders by cost. Source order is purely cosmetic.

---

### 11. The exercise's solution uses `goto-configuration!` to seed `:password` in session data. Could you also pass the password via the event itself?

- A) No — events can only carry their name, not data.
- B) No — guards can only read session data, not event data.
- C) Yes — call `(t/run-events! env {:name :submit :data {:password "secret"}})` and have the guard read `(get-in data [:_event :data :password])`. Both patterns are legal; the exercise picks the session-data path because it focuses on guards, not event-data plumbing.
- D) Yes, but only with `::sc/binding :late`.

---

### 12. The exercise's test creates *two* `let` blocks (with `env` and a separate inner `let` for `env2`). Why not reuse a single env?

- A) Because the chart's `:authenticated` is a final state — once entered, the chart can't be driven further. The fresh env is required to test the `wrong-password → :rejected → :retry` path independently.
- B) Because `start!` can't be called twice on the same env.
- C) Because `goto-configuration!` clears all event handlers.
- D) Because Clojure's `let` doesn't allow rebinding within scope.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **B** | Tutorial Step 1 — `[env data]`, env first. |
| 2 | **C** | Missing `:cond` = unconditionally true; first match in document order wins. Tutorial Step 2. |
| 3 | **B** | Clojure truthiness — only `false` and `nil` are falsy. Runbook "Try breaking it" #3. |
| 4 | **C** | Short-circuit: first matching+passing transition fires. Tutorial Step 2 ("A short-circuit, not a vote") + runbook Probe 4. |
| 5 | **B** | `:_event :data <key>` for the event payload. Tutorial Step 1 + runbook Probe 3. |
| 6 | **C** | `op/assign` returns a descriptor map; the runtime interprets it. Tutorial Step 3 + runbook Probe 1. |
| 7 | **B** | Test helper; sets data + configuration, bypasses entry actions. Tutorial Step 4. |
| 8 | **B** | `goto-configuration!` uses `configuration-for-states` which includes ancestors and `:ROOT`. See [gotchas.md #1](./gotchas.md). |
| 9 | **C** | Verified empirically — see [gotchas.md #2](./gotchas.md). The chart silently falls through; the exception is logged but doesn't crash. |
| 10 | **B** | Short-circuit semantics make ordering meaningful. Caveat: changing order changes which transition fires when both guards would pass — only swap if semantics allow. |
| 11 | **C** | Event data is in `data._event.data`. Both patterns are legal. Tutorial Step 1 + runbook Probe 3. |
| 12 | **A** | `:authenticated` has no outgoing transitions — it's effectively a final state. The fresh env is needed for the wrong-password scenario. (Note: not literally a `final` element — that's ex08 — just behaviorally "stuck.") |
