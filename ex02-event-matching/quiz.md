# Quiz — ex02: Hierarchical Event Names & Prefix Matching

13 multiple-choice questions covering the concepts established in [`tutorial.md`](./tutorial.md) and exercised in [`runbook.md`](./runbook.md). One correct answer per question. Answer key at the bottom.

If you got fewer than 10 right, re-read the section of the tutorial named in the "Quick reason" column.

---

### 1. The event name `:user.logged-in.via-sso` has how many segments, as seen by the library's matcher?

- A) 1 — keywords are atomic, the dots are decorative.
- B) 2 — segments split on `.` *and* on `-`.
- C) 3 — segments split only on `.`. Hyphens stay inside a segment.
- D) 4 — `user`, `logged`, `in`, `via-sso`.

---

### 2. A transition declares `(transition {:event :error :target :error-display})`. Which incoming events does it match?

- A) Only the literal event `:error`. To catch `:error.critical`, you'd need a separate transition.
- B) `:error` and any event whose name has `error` as a prefix on a segment-by-segment basis — including `:error.critical`, `:error.recoverable`, `:error.foo.bar`.
- C) Any event whose name *contains* the substring `error` anywhere, including `:my-error.x`.
- D) Only events that exactly match `:error` *or* exactly match `:error.*` (with literal `.*`).

---

### 3. The exercise chart has a `:event :error.critical` transition inside `:working` and a `:event :error` transition inside `:operational`. When `:error.critical` arrives while the chart is in `:working`, which transition fires?

- A) Both fire — the runtime checks every matching transition and runs all of them.
- B) The `:error → :error-display` transition, because the catch-all is "more general."
- C) The `:error.critical → :shutdown` transition. The ancestor walk starts at `:working` and stops at the first match it finds; `:operational`'s `:error` transition is never consulted.
- D) Neither — there's ambiguity, so the runtime drops the event.

---

### 4. The exercise chart is in `:working`. `:error.unknown` arrives. The chart ends up in `:error-display`. Why?

- A) The `:event :error.unknown` keyword is automatically aliased to `:event :error` when no exact handler exists.
- B) `:working` has no transition that matches `:error.unknown`. The ancestor walk continues to `:operational`, whose `:event :error` transition prefix-matches `:error.unknown` and fires.
- C) Because `:error-display` is alphabetically the first state whose name starts with "error," it's the default destination for unhandled error events.
- D) Because the chart has a global error-handler fallback that always routes to `:error-display`.

---

### 5. After `(t/start! env)` runs on the exercise chart, what does the configuration set contain?

- A) `#{:working}` only — `t/in?` reports the deepest active state.
- B) `#{:ROOT :operational :working}` — every ancestor including the chart root.
- C) `#{:operational :working}` — the active atomic state plus its compound ancestors, but **not** the auto-generated `:ROOT`.
- D) `#{:operational}` only — compound states subsume their children.

---

### 6. You query `(t/in? env :operational)` while the chart is in `:working`. What do you get and why?

- A) `false` — `:operational` is just a container; `t/in?` only reports atomic leaves.
- B) `true`, because the configuration set explicitly contains `:operational` (the compound state is active when one of its children is active). `t/in?` is a set-membership check.
- C) `true`, but only because `t/in?` walks up the ancestor chain. The configuration itself only contains `:working`.
- D) An error — `t/in?` can only be called with the ID of an atomic state.

---

### 7. A `transition` element is declared as a direct child of the chart root (alongside states). What happens?

- A) It works — top-level transitions are global and fire from any state.
- B) The `statechart` factory throws an `ex-info` complaining about an illegal top-level node. Transitions must be nested inside a state (or other element that legally contains them).
- C) The runtime ignores it silently.
- D) It works but only for namespaced events.

---

### 8. According to the library's *documented* matching behavior, does `(name-match? :error :errors)` return true?

- A) Yes — the documentation says "segment-by-segment prefix match," and `:error` is a string prefix of `:errors`.
- B) No — `:errors` has a single segment `["errors"]`, and the candidate's segment `["error"]` is not equal to it. By the documented rule, no match.
- C) The function throws because `:errors` isn't a valid event name.
- D) Yes — the matcher splits on any non-alphanumeric character.

---

### 9. In practice (empirically — not just per the docstring), does `(name-match? :error :errors)` return true?

- A) No, both per docs and in practice.
- B) Yes — the library's `name-match?` has an undocumented string-prefix fallback for candidates without namespaces. `":errors"` starts with `":error"`, so the fallback returns true even though the segment match would not.
- C) The behavior depends on the chart's `::sc/document-order` setting.
- D) Yes, because the matcher canonicalizes hyphens and `s` suffixes.

---

### 10. The catch-all `:event :error` transition is placed inside `:operational`, not at the chart root. If you moved it inside `:working` *before* the specific `:error.critical` and `:error.recoverable` transitions, what would happen to `:error.critical`?

- A) Nothing — placement and order don't affect which transition fires; specificity does.
- B) It would still route to `:shutdown`, because the runtime prefers the longer-match candidate.
- C) It would route to `:error-display`. Document order makes the `:event :error` transition the first match; the ancestor walk doesn't get to consult the more-specific transitions below it.
- D) The chart errors at construction time because two transitions can't match the same event.

---

### 11. Does `(name-match? :my/error :other/error)` return true?

- A) Yes — both keywords have the same name (`error`); namespaces are advisory.
- B) Yes — any qualified keyword matches any other qualified keyword with the same final segment.
- C) No — when the event has a namespace, the candidate's namespace must match exactly. `:my` and `:other` are different namespaces.
- D) Yes, but only when the chart has `:binding :late`.

---

### 12. The candidate `:error.*` is declared on a transition's `:event` key. What does the library do with the `.*`?

- A) It's a regex — the `.` matches any character, `*` is the Kleene star. The library invokes Java's regex engine on the event name.
- B) It's stripped before matching. `:error.*` is treated exactly the same as `:error`. The notation is a readability convenience for "this is a prefix matcher."
- C) It's required for prefix matching. Without `.*`, the candidate would only match the exact event `:error`.
- D) It causes a compile-time warning because `*` isn't a valid keyword character.

---

### 13. A transition is declared `(transition {:event [:user-cancel :timeout] :target :abandoned})`. What's the semantics of the vector?

- A) **AND** — the transition fires only when *both* `:user-cancel` and `:timeout` have been delivered.
- B) **OR** — the transition fires when *either* `:user-cancel` or `:timeout` arrives. Each candidate in the vector is checked independently against the incoming event using prefix matching.
- C) **Tuple** — the library treats `[:user-cancel :timeout]` as a single composite event name; only an exact-tuple event matches.
- D) **Sequential** — the transition fires only when `:user-cancel` arrives first, *then* `:timeout`.

---

## Answer key

| # | Answer | Quick reason |
| --- | --- | --- |
| 1 | **C** | Dots are segment separators; hyphens are not. Tutorial Step 1. |
| 2 | **B** | Prefix match on segments. Tutorial Step 2. |
| 3 | **C** | Ancestor walk + first match wins. Tutorial Step 3. |
| 4 | **B** | The ancestor walk falls through to `:operational` when `:working` has no match. Tutorial Step 3, end. |
| 5 | **C** | Compound state + active atomic, with `:ROOT` excluded. Tutorial Step 4 + ex01 review of the `:ROOT` rule. |
| 6 | **B** | `t/in?` is `(contains? configuration state-name)`, set membership. The configuration *contains* the compound, so the check returns true directly. Tutorial Step 4. |
| 7 | **B** | `statechart` validates top-level children against `#{:state :parallel :final :data-model :script}`. See `chart.cljc` (the `legal-node-types` set) and the "Common breakages" entry in the runbook. |
| 8 | **B** | Per the docstring, matching is segment-by-segment. Segments `["error"]` ≠ `["errors"]`. |
| 9 | **B** | Empirically (Probe 3 in the runbook), `name-match?` has a string-prefix fallback for non-namespaced candidates. Tutorial Step 6. |
| 10 | **C** | Document order resolves ties within a state. The first-declared transition with a matching event wins; specificity doesn't sort. Runbook "Try breaking it" #3. |
| 11 | **C** | Namespaces are strict; both keyword's namespaces must be equal. Tutorial Step 5. |
| 12 | **B** | The library strips `.*` from the candidate (events.cljc, `name-match?`, line 28). Tutorial Step 2 ("Wildcards"). |
| 13 | **B** | `:event` accepts a vector; the matcher returns true if *any* element matches. See `name-match?` in events.cljc (the vector branch). Tutorial Step 2 (`:event` can be a vector). |
