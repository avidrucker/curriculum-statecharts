# Questions for Tony Kay (library author)

Open questions about `com.fulcrologic/statecharts` 1.2.25 that **cannot be resolved by reading source, running probes, or watching the existing talks**. They're about *intent*, *contract*, or *history* — things only the author can definitively answer.

See [`CLAUDE.md`](./CLAUDE.md) → "`ask_tony.md` — questions for the library author" for the rationale and format.

This file is append-only during authoring; resolved questions get folded back into the relevant docs and removed from here in periodic cleanup passes.

---

## Open

### Q1: Is the string-prefix fallback in `name-match?` intentional or a bug?

**Where this came up.** ex02 tutorial Step 6, ex02 glossary `string-prefix fallback (footgun)`, ex02 gotchas #1.

**What I verified empirically.** `events.cljc:25`'s `name-match?` has two branches for non-namespaced candidates: a documented segment-prefix match, and an *undocumented* string-prefix fallback via `(str/starts-with? (str event-name) (str/replace (str candidate) #"\.\*$" ""))`. The fallback causes:

- `(name-match? :error :errors) => true`  (string `":errors"` starts with `":error"`)
- `(name-match? :error :error-thing) => true`
- `(name-match? :error.crit :error.critical) => true`  (partial-segment match via string)

These results contradict the docstring's "segment-by-segment basis" framing.

**What I can't tell from the code.** Is the string fallback (a) an intentional behavior for some use case I'm not seeing, (b) a backwards-compat artifact of an earlier matching strategy, (c) a known bug that's been deferred, or (d) something the maintainer wants to remove in a future major version?

**Provisional answer in the docs.** Documented as a "footgun" with defensive guidance ("don't name events such that one is a string-prefix of another"). The framing is intentionally non-committal about whether it's a bug — if it's by design, we'd want to soften the gotcha; if it's a bug due-for-removal, we'd want to flag that it might change.

---

### Q2: Is `t/start!`-as-reset a supported behavior?

**Where this came up.** ex01 glossary `t/start!`, ex01 gotchas #6, ex01 tutorial Step 1 (mentions the reset behavior).

**What I verified empirically.** Calling `(t/start! env)` on an env that's already running:
- Does not throw.
- Resets the chart back to its initial configuration (the second `start!` overwrites the working memory).

**What I can't tell from the code.** Is this an *intentional* contract — i.e., users can rely on `start!` as a reset operation in future versions — or is it incidental behavior that might break in 2.x? The exercise tests use a fresh-env-per-scenario idiom, which suggests fresh-env is preferred, but it doesn't tell me whether reset-via-restart is forbidden, discouraged-but-supported, or actively encouraged for some use case.

**Provisional answer in the docs.** Currently the curriculum says "works as a reset, but the fresh-env pattern is the convention." If `start!`-as-reset is supported, we could mention it more positively. If it's incidental, we should warn against relying on it.

---

### Q3: Why is `:ROOT` excluded from the configuration set?

**Where this came up.** Top-level glossary `configuration`, ex01 glossary `chart root (:ROOT)`, ex01 gotchas #3.

**What I verified empirically.** After `(t/start! env)` on a flat chart, the configuration set is `#{:red}` (or whichever atomic state is active). After a nested chart enters `:working` (child of `:operational`), the configuration is `#{:operational :working}`. In neither case is `:ROOT` in the set — `(t/in? env :ROOT) => false` even when the chart is clearly running.

**What I can't tell from the code.** SCXML's spec includes the document root in some configuration-related calculations. Is the library's exclusion of `:ROOT` from `::sc/configuration` (a) faithful to SCXML's distinction between the chart root and "real" states, (b) a pragmatic choice (e.g., to make `t/in?` more useful), or (c) something with subtler semantics that I'm missing?

**Provisional answer in the docs.** Currently framed as "the root is the *container*, not a state the chart is *in*." If there's a deeper SCXML-aligned reason, we'd want to surface it. If it's pragmatic, the current framing is fine.

---

### Q4: Why is the validator's "Cannot register invalid chart" error opaque?

**Where this came up.** ex02 runbook Common Breakages (new entry), ex02 gotchas #7.

**What I verified empirically.** A chart with a `:target` referring to a nonexistent state ID throws `"Cannot register invalid chart"` at registration time. The exception does *not* name the bad reference, the bad transition, or any other diagnostic detail.

**What I can't tell from the code.** Is the opaque error (a) a known UX gap that's planned to improve, (b) a side-effect of the validator producing a problems-vector that the registration code throws away, or (c) intentional (some philosophical reason)? Is there a programmatic API that returns the validator's problems list directly, so we can teach the learner to call it for diagnosis?

**Provisional answer in the docs.** Currently the curriculum says "the error is opaque on purpose, not a curriculum oversight" and gives a grep-based diagnostic recipe. If there's a programmatic way to surface the validator's problem list, we'd want to teach that instead of the grep.

---

### Q5: Are auto-generated `:initialNN` and `:state-NN` IDs stable across runs?

**Where this came up.** ex01 gotcha #2 (missing `:id` on a state), ex01 glossary `chart root (:ROOT)` (the auto-initial element).

**What I verified empirically.** A `state` element without an explicit `:id` gets an ID like `:state25839`. The chart factory also auto-inserts an `initial` element with IDs like `:initial17446` or `:initial17441`. The numbers *vary across runs of my probe* — they're not deterministic per chart-definition.

**What I can't tell from the code.** Is the per-run variation intentional (e.g., a counter that resets per JVM invocation, with the variation tolerated as harmless), or a hash that's expected to be stable per chart but happens to differ for some reason I'm not tracking (e.g., because I built the chart in different JVMs)? Is there a way to make these IDs deterministic for testing — so a test that asserts on chart structure isn't fragile?

**Provisional answer in the docs.** Currently the curriculum just says "library bookkeeping; you don't reference these directly." If there's a way to stabilize them or a guarantee about their stability, we'd want to surface it.

---

### Q7: Is the guard-throws-becomes-false behavior intentional or incidental?

**Where this came up.** ex03 gotchas #2, ex03 quiz Q9, ex03 runbook "Common breakages" entry.

**What I verified empirically.** A guard that throws does not propagate the exception out of `(t/run-events! env :submit)`. The runtime catches the exception (logged at DEBUG level via `condition-match` / `run-expression!`), treats the guard as having returned `false`, and falls through to the next matching transition. The chart continues running normally.

```clojure
(transition {:event :submit :cond (fn [_ _] (throw (ex-info "boom" {}))) :target :authenticated})
(transition {:event :submit :target :rejected})
;; (t/run-events! env :submit) → no throw, chart lands in :rejected
```

**What I can't tell from the code.** Is this catch-and-suppress (a) an intentional resilience feature (charts should be robust to buggy guards), (b) a development-convenience that ought to switch to throwing in some "strict" mode, (c) historical behavior inherited from an earlier processor, or (d) a known footgun that should ideally be configurable per chart?

**Provisional answer in the docs.** The gotcha documents the behavior as a footgun with defensive guidance ("keep guards pure," "wrap in try/println when debugging"). The framing is non-committal about whether it's a feature or a bug. If it's an intentional resilience feature, we'd reframe more positively; if it's a known footgun with a planned fix, we'd say so.

---

### Q8: Why does `goto-configuration!` include `:ROOT` in the configuration set when `start!` excludes it?

**Where this came up.** ex03 gotchas #1, ex03 quiz Q8, ex03 tutorial Step 4.

**What I verified empirically.** Same chart, two ways to land in the same logical state, two different configuration sets:

```clojure
;; via start!
(t/start! env)
(t/in? env :ROOT)   ; → false   (config = #{:a})

;; via goto-configuration!
(t/start! env)
(t/goto-configuration! env [] #{:a})
(t/in? env :ROOT)   ; → true    (config = #{:ROOT :a})
```

After subsequent `(t/run-events! env :go)`, `:ROOT` *persists* in the config of the goto-config'd env. The asymmetry stems from `goto-configuration!` calling `configuration-for-states`, which the docstring describes as "the union of the set of `states` and their proper ancestors" — and the chart root counts as a proper ancestor by that definition. The runtime's `start!` lifecycle excludes the root via different code paths (probably the SCXML algorithm in `algorithms.v20150901-impl`).

**What I can't tell from the code.** Is the asymmetry (a) intentional — `goto-configuration!` is a test helper and its inclusion of `:ROOT` reflects "the chart is really running here, including in its root context," (b) an oversight where one of the two code paths should be updated to match the other, or (c) reflecting a deeper SCXML distinction between "active configuration during normal execution" vs "configuration valid for a test setup"?

**Provisional answer in the docs.** The gotcha names the asymmetry, warns against writing `t/in?` checks that depend on `:ROOT`'s presence/absence, and recommends checking by which entry path was used. If the asymmetry is intentional, the gotcha framing is fine. If one of the two paths is wrong, we'd update the curriculum's foundational claim about `:ROOT` exclusion.

This question subsumes Q3 (which asked "why is `:ROOT` excluded from the normal-lifecycle configuration"). Q3 remains relevant as the conceptual question; Q8 is the more specific "and why the asymmetry."

---

### Q9: What expression types does `:cond` accept besides functions?

**Where this came up.** ex03 glossary `:cond (guard)` entry — the library's `transition` docstring says `:cond` is "an expression that must be true for this transition to be enabled. See execution model." The curriculum currently shows only function literals, but "expression" implies more is supported.

**What I verified empirically.** Function literals work: `(fn [env data] (= (:password data) "secret"))`. I have not tested whether the library accepts:

- Plain values: `:cond true` / `:cond false` / `:cond nil`
- Vars: `:cond #'my-guard-fn`
- Symbols: `:cond 'my-guard-fn`
- Maps that the execution model interprets specially
- Non-fn data structures (in case the execution model has a special interpretation)

**What I can't tell from the code without deeper investigation.** I haven't read `com.fulcrologic.statecharts.execution-model.lambda` (the default execution model) carefully enough to enumerate the supported expression forms. This is *probably* answerable by reading the source, but the answer's *intended use* — what should the curriculum recommend? — is the author's call.

**Provisional answer in the docs.** The glossary entry currently says "in ex03 the expression is always a function literal; the library's execution model supports other forms (see `ask_tony.md`)." If only fn literals are supported in practice (because the test harness uses the lambda execution model exclusively), I'd remove the hedge. If other forms are real and recommended, I'd document them.

---

### Q10: Cross-region transitions — is "source state stays in the configuration" the intended SCXML behavior or a library quirk?

**Where this came up.** ex04 gotchas #6, ex04 glossary `cross-region transition`, ex04 runbook "Try breaking it" #2.

**What I verified empirically.** A transition from `:ra-1` (in region `:ra` of parallel `:par`) targeting `:rb-2` (in region `:rb`) produces this configuration shift:

```
before :cross  →  #{:par :ra :ra-1 :rb :rb-1}
after  :cross  →  #{:par :ra :ra-1 :rb :rb-2}
```

The source state `:ra-1` remains in the configuration. The target `:rb-2` becomes active in `:rb`. In-region transitions exit their source state cleanly; cross-region transitions don't.

**What I can't tell from the code.** Two possibilities:

(a) **Intended**: SCXML's LCA-based exit-set computation says the LCA of a cross-region transition is the parallel, so the exit set doesn't include the source's region — leaving the source state active is *structurally correct*. The library follows the spec.

(b) **Quirk**: the library's interpretation produces a configuration that's *technically valid* per the LCA rule but operationally surprising. The "correct" behavior arguably would be to re-initialize the source's region (or treat cross-region transitions as a chart-level error).

Either way, the curriculum currently warns against the pattern. But if (a) is the case, we'd frame it as "advanced SCXML, here's when you'd actually use it"; if (b), we'd frame more strongly as "the library will let you write this but the behavior is undefined-in-practice."

**Provisional answer in the docs.** Documented as "don't write cross-region transitions" with the empirical configuration outcome shown. Framing is non-committal about whether it's a feature or a quirk.

---

### Q6: Is `(def scxml statechart)` an actively-supported alias, or a legacy holdover?

**Where this came up.** Top-level glossary `statechart` (mentions the alias).

**What I verified empirically.** `chart.cljc` defines `(def scxml "Alias for statechart." statechart)`. Calling `scxml` works identically to calling `statechart`.

**What I can't tell from the code.** Is `scxml` (a) recommended for users coming from XML SCXML who want the familiar naming, (b) deprecated but kept for back-compat, or (c) just an undocumented synonym that came along for the ride? The curriculum currently mentions it neutrally; we'd want to recommend or de-emphasize based on the author's intent.

**Provisional answer in the docs.** "The library also exposes `scxml` as an alias for `statechart`, for readers who prefer the SCXML naming." Neutral. If `scxml` is the preferred form in some context, we'd say so.

---

## Resolved

*(none yet — questions move here with the author's answer + the doc-set update that incorporated it)*
