# Gotchas — what they are and how to author them

A per-exercise `gotchas.md` is the **fifth doc type** in this curriculum, after [`tutorial.md`](./overview.md), [`runbook.md`](./overview.md), [`quiz.md`](./overview.md), and [`glossary.md`](./overview.md). It catalogues the **silent failures, hidden behaviors, and easy-to-miss configurations** the learner is likely to encounter — concentrated in one place so a stuck learner has a single doc to scan.

This file explains what belongs in a `gotchas.md`, how to structure entries, and how it differs from the other doc types.

---

## What "gotcha" means in this curriculum

A gotcha is a thing that **bites you silently or surprisingly**. Three criteria:

1. **The code compiles / the chart loads / the test runs.** No syntax error, no exception. The runner produces output. (Loud failures — `Unable to resolve symbol`, `Illegal top-level node` — belong in the runbook's "Common breakages," not here.)
2. **The behavior is surprising to a reader who's followed the tutorial.** Either it does something the docstring doesn't promise, or the docstring is technically accurate but the intuitive reading is wrong, or the library does something unrelated to what the learner thought they were doing.
3. **The defense is non-obvious.** If "read the docs more carefully" fixes it, it's not a gotcha — it's just a missed read. A gotcha has a defensive practice the docs don't surface clearly.

Three categories tend to recur:

| Category | Examples in this library |
| --- | --- |
| **Silent failures** | Missing `:id` auto-assigns one; `(t/in? env :typo)` returns false instead of erroring; unknown events are silently dropped |
| **Hidden behaviors** | The string-prefix fallback in `name-match?`; auto-generated `:initial` elements at every compound level; `:ROOT` exclusion from the configuration set |
| **Easy-to-miss config** | Forgetting the `elements` require; forgetting the chart options map `{}`; one-way chart paths needing fresh envs |

---

## How `gotchas.md` differs from other doc types

| Doc type | Question it answers | When the learner reads it |
| --- | --- | --- |
| `tutorial.md` | "Why does this work this way?" | Before solving |
| `runbook.md` | "How do I make this run, and where does it break?" | While solving |
| `quiz.md` | "Did I internalize the concepts?" | After solving |
| `glossary.md` | "What does this term mean?" | On demand, throughout |
| **`gotchas.md`** | **"What might bite me even when nothing looks wrong?"** | **When confused; when the tests pass but something feels off; before working on real code** |

Two overlaps worth knowing:

- **vs. runbook's "Common breakages"**: those entries are *symptom-indexed* and *tied to test failures* (`The test fails with X — here's the bisection`). Gotchas covers things where there's no test failure to bisect — the bite is silent or it bites you weeks later in a different codebase.
- **vs. tutorial's "Common misconception" callouts**: those are inline corrections to the mental model the tutorial is currently building. Gotchas collects them — plus the ones the tutorial didn't have space for — into a scannable reference.

There's deliberate redundancy. A learner who reads the tutorial straight through encounters misconceptions inline. A learner debugging at 11pm wants one doc to scan. Both work.

---

## Per-entry structure

Each gotcha is a short section with four fields:

```markdown
### Short, evocative title (the surprise)

**What you'll see.** One or two sentences. Concrete — a code snippet, a test result, a REPL output the learner would recognize.

**What's really happening.** One to three sentences. Names the mechanism. Cites library source or empirical probe.

**Defense.** One to two sentences. Concrete defensive practice — a habit, a check, an alternative idiom.

*(Optional)* **See also.** Links to relevant tutorial step, glossary entry, runbook section.
```

**Length per entry: 4–8 lines.** Long enough to ground the gotcha; short enough that the doc remains scannable. If an entry needs more, ask whether the content actually belongs in the tutorial or glossary.

**Length of the whole `gotchas.md`: 6–12 entries.** Fewer than 6 means the exercise probably hasn't surfaced enough surprises (or you haven't dug for them). More than 12 means it's drifting into a glossary or tutorial.

---

## Sourcing entries

Each entry must be **empirically verified**, not theorized. The way you verify:

1. **Reproduce the surprise.** Write a small REPL probe that demonstrates the unexpected behavior.
2. **Cite the source.** If the surprise stems from library code, link to the file/line. If it stems from an undocumented behavior, note that explicitly.
3. **Confirm the defense works.** If you write "do X instead," verify X actually avoids the bite.

The bar is the same as for runbook "Common breakages": **theoretical gotchas drift from what learners actually experience.** Skip them.

Some good sources for gotcha entries:

- **Your own writing process.** Every empirical surprise you hit while authoring `tutorial.md` or `runbook.md` is a candidate gotcha. (The string-prefix fallback in ex02 was found this way.)
- **The review pass.** Things you got wrong on the first authoring pass and corrected — those *were* gotchas for you, so they probably are for learners too.
- **Library source quirks.** Reading the library code and noticing "the docstring says X but the code does X+Y" — that's a gotcha.
- **Real questions from real learners.** When someone asks "why doesn't this work?" and the answer is non-obvious, that's a gotcha source.

Skip:

- **Things the runbook already documents as "Common breakages."** Don't duplicate.
- **Things the tutorial has as a "Common misconception."** Reference, don't repeat.
- **General Clojure surprises.** This is curriculum about the statecharts library, not about REPL workflow basics. Don't write a gotcha for "you forgot to `:reload`" — that's general Clojure tooling.

---

## Look-ahead policy

The same look-ahead rule that applies to the rest of the curriculum applies here: **a gotcha belongs to the exercise where the concept that it bites on is first introduced.**

Examples:

- The string-prefix matching footgun → ex02 (event matching introduced there)
- Guard expressions that are called multiple times per event → ex03 (guards)
- Parallel-region event broadcasting surprises → ex04 (parallel states)
- History state's "default target" subtleties → ex06 (history)

If a gotcha cuts across multiple exercises (e.g., REPL caching), pick the *earliest* exercise where it's likely to bite and put it there. Subsequent exercises can link back via `[[link]]`.

---

## Cross-references

Each gotcha entry should link out where useful:

- `[[term]]` to the glossary (per-exercise or top-level) for vocabulary.
- `[tutorial Step N](./tutorial.md)` for the conceptual grounding.
- `[runbook Step N](./runbook.md)` or `[runbook 'Common breakages'](./runbook.md)` for the operational pair.

Don't over-cross-reference. If three of the four fields point at the same tutorial step, the gotcha probably belongs *in* the tutorial as an inline callout, not as its own gotcha.

---

## Pre-authoring checklist

Before drafting a module's `gotchas.md`:

1. **Re-read your own [`tutorial.md`](./overview.md), [`runbook.md`](./overview.md), and [`glossary.md`](./overview.md) for the module.** Note every "Common misconception" callout and "Common breakages" entry. The gotchas should *complement* these, not duplicate them.
2. **Re-read the [`overview.md`](./overview.md) exercise map.** Note what's coming in later exercises so you don't write a gotcha that belongs in ex05 inside ex02's file.
3. **List candidate gotchas.** Brainstorm in scratch — REPL probes you ran, surprises you hit, library quirks you stumbled on. Aim for 10–15 candidates, then trim to 6–12 entries.
4. **Verify each candidate empirically.** A short REPL session per gotcha; capture the output. If it doesn't reproduce, drop it.
5. **Order by likelihood-of-biting.** First entries should be the highest-frequency surprises; later entries can be edge cases and quirks. The learner who scans the first three should already get value.

---

## Anti-patterns to catch in review

- **Theoretical gotchas.** "If you did X, then Y might happen" without an empirical reproduction. Cut.
- **Gotchas that are really tutorial gaps.** If the tutorial should have covered this concept directly, fix the tutorial — don't paper over with a gotcha.
- **Gotchas that are really runbook bugs.** If the symptom comes from a specific test failure, it belongs in "Common breakages."
- **Wall-of-text entries.** Each entry should be readable in 30 seconds. If it isn't, the framing is off.
- **Stale gotchas.** When the library version changes, re-verify each entry. Drop or update ones that no longer reproduce.

---

## When in doubt

Three questions:

- **Would a learner who solved this exercise still get bitten by this six months later?** If yes, it's a gotcha. If no, it's a tutorial moment.
- **Can the learner verify the gotcha themselves in 30 seconds at the REPL?** If yes, include the probe. If no, the explanation needs more grounding.
- **Is the defense memorable?** A gotcha without a clear defense is just a curiosity. If you can't name the defense in one sentence, the entry isn't ready.

Gotchas are the doc type that ages worst — library behaviors change, conventions evolve, the things-that-bite mature. Treat the file as a living document; re-verify on major library upgrades, drop entries whose biting power has waned.
