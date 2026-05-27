# Issues — ex03 (deferred items to investigate)

Items discovered during ex03 authoring that weren't resolved in the first-pass session. These are *empirical* questions (mostly answerable with more probes) — distinct from the *intent / contract* questions in [`../ask_tony.md`](../ask_tony.md), which can only be resolved by the library author.

Format:

```markdown
### Issue title (one-line summary)

**Discovered while.** Where this surfaced during authoring.
**What we don't know.** The specific gap.
**How to investigate.** Concrete next step — typically a REPL probe.
**Impact on the docs.** What the curriculum currently says, and what might need updating once resolved.
```

When an issue is resolved, fold the answer into the relevant docs and either mark the entry **Resolved: <date> — <one-line outcome>** or remove it.

---

## Open

### Issue 1: Ancestor walk + failed guards — does the walk continue?

**Discovered while.** Writing the ex03 runbook / gotchas. ex02 established the ancestor walk rule (event arrives → check active state's transitions first → if no match, check ancestor's transitions → ...). ex03 introduced guards. The composition — "inner state has a transition matching the event but its guard fails; ancestor has a matching transition" — wasn't probed.

**What we don't know.** Concretely, given this chart:

```clojure
(state {:id :outer}
  (transition {:event :go :target :elsewhere})            ; ancestor transition, no guard
  (state {:id :inner}
    (transition {:event :go :cond (fn [_ _] false)
                 :target :other})))                       ; inner transition, guard fails
```

When the chart is in `:inner` and `:go` arrives:

- Hypothesis A: The runtime tries `:inner`'s transition; the guard fails; falls through to `:outer`'s transition; fires; chart enters `:elsewhere`.
- Hypothesis B: The runtime tries `:inner`'s transition; the guard fails; *no further walk happens* (the transition "matched the event" so the walk stops, even though the guard rejected); chart stays in `:inner`.

The SCXML spec leans toward Hypothesis A (per the "select-transitions" algorithm: a transition is only "enabled" if both event matches AND guard passes; non-enabled transitions don't block the walk). The library probably follows SCXML, but it hasn't been verified.

**How to investigate.** Single REPL probe: build the chart above, fire `:go`, check `(t/in? env :elsewhere)`. 5 lines.

**Impact on the docs.** ex03 tutorial Step 2 currently describes the within-state short-circuit ("first match wins"). It doesn't address the across-state case. ex02 tutorial Step 3 implies Hypothesis A but doesn't say so explicitly. Either way, a small explicit confirmation either in ex03's tutorial or as a glossary note for `[[ancestor-walk]]` would close the gap.

---

### Issue 2: What expression types does `:cond` accept besides function literals?

**Discovered while.** Writing the ex03 glossary entry for `:cond (guard)`. The `transition` element's docstring says `:cond` is "an expression that must be true for this transition to be enabled. See execution model." Every example in the curriculum and in the library's test suite uses a function literal, but "expression" implies the execution model accepts more.

**What we don't know.** Whether any of these work as `:cond` values:

- A plain boolean: `:cond true`, `:cond false`
- A keyword: `:cond :some-key` (treat as data-model lookup?)
- A vector / list / quoted form: `:cond [:= [:data :password] "secret"]`
- A var: `:cond #'my-guard-fn`
- A symbol: `:cond 'my-guard-fn`

Per the library's namespaces, the default execution model is in `com.fulcrologic.statecharts.execution-model.lambda` (the lambda model). Reading that source would answer most of this. There may also be alternative execution models (the library is protocol-driven on this axis).

**How to investigate.** Two-part:

1. Read `com/fulcrologic/statecharts/execution_model/lambda.cljc` from the library jar to enumerate which expression shapes are dispatched on (likely via `cond` / `case` on the value's type or marker).
2. REPL-probe each candidate shape to confirm.

The result also feeds [`ask_tony.md` Q9](../ask_tony.md), which asks the *curriculum-design* question (should we recommend any alternative shapes?). Issue 2 is the *factual* question (what shapes work?).

**Impact on the docs.** The ex03 glossary currently hedges with "in ex03 the expression is always a function literal; the library's execution model supports other forms." If only function literals work in practice, drop the hedge. If others work, mention briefly and refer to ex03c or wherever convenience shapes are introduced.

---

---

## Resolved

### Issue 1: Ancestor walk + failed guards — does the walk continue?

**Resolved: 2026-05-27 — yes, the walk continues.** Probe in ex03b's authoring session (probe P9) confirmed it. With the chart:

```clojure
(state {:id :outer}
  (transition {:event :go :target :elsewhere})
  (state {:id :inner}
    (transition {:event :go :cond (fn [_ _] false) :target :other})))
```

When the chart is in `:inner` and `:go` arrives: `:inner`'s transition is skipped (guard false), the walk continues to `:outer`, `:outer`'s transition fires, the chart enters `:elsewhere`. Hypothesis A confirmed.

A brief sentence has been added to the ex03 tutorial Step 2 noting this. No further action needed.

### Issue 3: Does `goto-configuration!` preserve history values?

**Resolved: 2026-05-27 (curriculum review pass) — yes, preserved.** Probe constructed a chart with deep history, established a recorded value (`{:h #{:b}}`), called `(t/goto-configuration! env [] #{:main})` to jump elsewhere, then verified:

1. The `::sc/history-value` in working memory was unchanged across the `goto-configuration!` call.
2. Re-entering via the history node (`:open :target :h`) correctly restored `:b` (not the default `:a`).

The library docstring claim ("Previously recorded state history is UNAFFECTED") holds empirically. No further action; the ex03 glossary's quote of the docstring is accurate.
