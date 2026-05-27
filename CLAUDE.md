# CLAUDE.md — Authoring guidance for `curriculum-statecharts/`

This file directs Claude (or any human author) when writing, revising, or reviewing curriculum content in this repo. Read it fully before authoring anything new.

Adapted from `curriculum-onboarding-rad-project/CLAUDE.md`, narrowed to the four-doc-per-module structure used here.

---

## Prime directive

**Never publish curriculum content about the `com.fulcrologic/statecharts` API without verifying it against the actual library source or the worked solution files in the sibling [`statechart-exercises`](../statechart-exercises) repo first.** The library is opinionated, the names matter, and several plausible-sounding identifiers (`states/state`, `transition/on`, `:event/match`) are *not* what the library ships. The way you correct a wrong claim later is by reading the source you should have read first.

---

## Source-of-truth verification order

When a curriculum doc disagrees with reality, the resolution order is:

1. The **library source** — the `com.fulcrologic/statecharts-1.2.25.jar` in `~/.m2`. Use:

   ```bash
   SC_JAR=$(find ~/.m2 -name "statecharts-1.2.25.jar" | head -1)
   unzip -l "$SC_JAR" | grep -i <symbol>
   unzip -p "$SC_JAR" "com/fulcrologic/statecharts/<file>.cljc" | grep -n "<symbol>"
   ```

2. The **worked solution files** in `../statechart-exercises/src/solutions/`. Each `ex0N_*.cljc` is a known-passing implementation; the `deftest` blocks are the executable spec.

3. **Tony Kay's talks** in `../fulcro-statecharts-talks/`. Useful for intent and motivating examples; not authoritative when the library has evolved past a talk.

4. The exercise file **docstrings** in `../statechart-exercises/src/exercises/`. Useful for topic framing; *not* always reliable for low-level facts (e.g., the `Run with: clj -X:test` claim in those docstrings is incorrect — the actual invocation is `clojure -M:test`).

If you find a discrepancy at any level, prefer the higher source and log the lower-source error so it can be fed back upstream.

---

## Authoring order (mandatory for new modules)

When creating a new module's curriculum set from scratch, draft the four docs in this order — **not** alphabetical, **not** runbook-first:

1. **Verify the solution.** Read the exercise stub and the solution side by side. Run the solution against the test suite — `clojure -M:test --focus exercises.ex0N-<topic>/<test-var>` — and confirm green. *The solved exercise is the spine of every other doc.*
2. **`tutorial.md`** — the conceptual narrative. Each tutorial section maps to one or more chart elements the runbook will then build, and explains the *why* before the runbook's *how*.
3. **`runbook.md`** — the operational pass. By now you know what the learner types and in what order, and the tutorial has established the *why* the runbook can lean on.
4. **`quiz.md`** — questions probe the concepts the tutorial established and the runbook exercised. Distractors reflect realistic misconceptions, not filler.
5. **`glossary.md`** — last. Every term that appeared in the other three has earned its place; the 0–9 [[criticality]] rating is informed by which terms turned out to be load-bearing.

### Why this order

Authoring the docs in any other order risks **inventing API surface that does not exist**. The lesson from the source curriculum: a runbook drafted with controls named `::form/new` (assumed to be a RAD built-in but actually invented) propagated the same error through tutorial, quiz, and glossary, all of which required separate corrections. Starting from a verified, runnable solution means every downstream doc is downstream of facts.

---

## Per-doc conventions

### `tutorial.md`

- **Top: "The story we're building"** — what the chart represents in the real world (a traffic light, a login gate, a checkout flow) and what the test assertions will prove.
- **Numbered steps** — each step introduces one cohesive concept, *concept first, mechanics second*. The reader should be able to explain the concept in plain English before they see the Clojure form.
- **`> Common misconception` callouts** — inline, named explicitly. When the natural learner intuition is wrong, name the wrong intuition before correcting it. (Example: "guards are tried in priority order" — they're not, they're tried in document order.) Source these from real quiz misses, real questions a learner has asked, real bugs that have shipped.
- **Cross-reference the glossary** with `[[term]]` syntax for high-criticality terms; reference the exercise file by name.
- **Close with "What you should be able to explain"** — a 3–5 bullet list naming the conceptual bar the quiz will test. Each bullet ends with the chart element or behavior that grounds it.
- **Target length:** 200–500 lines. Shorter for early modules; longer for ex04 (parallel) and ex09 (invocations).

### `runbook.md`

The operational pass — bridges "I understand the concept" to "I can make the test go green without guessing." For a unit-test-based exercise set, the runbook shape is:

1. **Opener** (required) — one paragraph: what the learner will have built when the test suite goes green, plus estimated time-to-complete.
2. **Prerequisites** (required) — one short paragraph naming the prior exercise(s) the reader should have completed and a pointer to [`setup.md`](./setup.md). Don't repeat setup.
3. **Read the exercise file first** (required) — quote the docstring, point at the `TODO` comment, list the namespaces it requires. Names the 1–2 library namespaces this exercise relies on.
4. **REPL warm-up** (required) — copy-paste a session that loads the exercise namespace and runs the (still-failing) tests. Locks in the diagnostic loop the rest of the runbook leans on.
5. **Build the chart, step by step** (required) — numbered build steps. Each step adds one chart element (a state, a transition, a guard) and re-runs the relevant test, with the *expected* failure-becomes-pass output in a *separate fenced block*. The numbered-subsection pattern (3.1, 3.2, …) handles multi-part steps.
6. **Try this (REPL experiments)** (recommended, 2–4 entries) — self-contained probes that build the debugger's reflex. Each experiment is framed as a diagnostic recipe, not curiosity-driven play: "**Scenario.** Your chart starts in the wrong state. **The probe.** Run `(t/in? env :red)` *before* `(t/start! env)`. **What this tells you.** …"
7. **Try breaking it** (recommended, 1–3 entries) — deliberate edits to the *finished* solution. Each entry names the file path as a repo-relative string (`src/exercises/ex0N_*.cljc`), shows the edit, predicts the symptom, and ends with an explicit `**Restore the X.**` instruction.
8. **Common breakages** (required, symptom-indexed) — entries named by what the learner *observes*, not by what code is broken. Body walks the bisection.
9. **What success looks like** (required) — a 5–8 item checklist of verifiable outcomes, closing with a one-breath synthesis (~25–40 words) the learner narrates back.

#### The four load-bearing conventions

These four, applied together, are what make a runbook function as a runbook (not as "tutorial with code samples"):

1. **Terminal labels everywhere.** Every code block names *where it runs* before the block: "**REPL (Terminal 1, exercise repo root).**" / "**Shell, exercise repo root.**" Use a labels table at the top if the runbook mixes terminals. (For most statechart exercises there's only one terminal — the REPL — so this collapses to a single up-front label.)
2. **Reusable `def`s, never `<placeholder>` substitution.** If a step binds `env` once, every subsequent block references `env` by name. The learner pastes verbatim. Patterns like `(t/in? env <state-keyword>)` force hand-editing and turn the runbook into a tutorial. Don't.
3. **Separate executable code from expected results.** **Never mix `;; =>` inline with executable code.** Always: one sentence of prose stating what's being checked → one fenced block (executable, no `;;` lines) → an "Expected:" line → a separate fenced block showing the result shape.
4. **Symptom-indexed diagnostics.** "Common breakages" entries are named by what the learner observes ("The test fails with `No matching transition`"), not by what code is broken. The body then walks the bisection — each bullet is a probe the learner can run, not a hypothesis to evaluate by reading code.

#### Length guidance

| Exercise density | Target lines |
| --- | --- |
| Thin (ex01, ex02, ex07) | 200–300 |
| Typical (ex03, ex03b, ex03c, ex05, ex06, ex08) | 300–450 |
| Dense (ex04 parallel, ex09 invocations) | 400–600 |

If a runbook is over 650 lines, ask whether content has drifted in from `tutorial.md` or `glossary.md`. The runbook is operational; if it's running long, something else is doing tutorial-shaped work.

### `quiz.md`

- **Length: as many MCQs as the exercise warrants, up to 15.** Some exercises (ex01: compound states) only need 6–8; the conceptually densest (ex04 parallel, ex09 invocations) may earn the full 15. Going under 6 means the exercise probably isn't being probed enough; going over 15 means the quiz is doing what `glossary.md` or `tutorial.md` should be doing.
- **One correct answer per question.** Multiple-choice, four to five options.
- **Distractors must be substantive.** If the correct option is three sentences and the others are five-word phrases, learners pattern-match the answer without reading the question. Make each distractor a real misconception a learner *would* have. Source them from quiz post-mortems, mentor anecdotes, the misconception callouts in `tutorial.md`.
- **Answer key at the bottom as a markdown table:** `# | Answer | Quick reason`. The "Quick reason" column is one sentence — enough that a learner who got it wrong knows whether to re-read the tutorial or the runbook.

### `glossary.md`

- **Per-exercise glossaries are additive to the top-level [`glossary.md`](./glossary.md).** They cover terms first introduced (or first deeply covered) in that exercise; cross-cutting terms ("state," "event," "transition") live at the top level and are referenced with `[[term]]` links.
- **Each entry: plain-language definition first, cross-references second.** Don't lead with the formal definition; lead with the working intuition, then refine.
- **0–9 criticality rating on every entry.** Use the scale anchored in the top-level glossary's intro. A 9 means "the learner cannot move past this exercise without it"; a 0–2 means "trivia worth knowing for context."
- **Length: 8–20 entries per exercise.** Sparser than the top-level glossary, denser than a quiz answer key.

---

## Pre-authoring research checklist

Before drafting **any** doc for a module, complete this:

1. **Read the exercise file in full** — both `src/exercises/ex0N_*.cljc` and `src/solutions/ex0N_*.cljc`. Don't skim. The docstring, the `TODO` comments, the test assertions, and the solution's chart structure are the spine.
2. **Run the solution.** `clojure -M:test --focus exercises.ex0N-<topic>/<test-var-name>` from the exercise repo root. Confirm green. Capture the actual summary line text (`1 tests, N assertions, M failures.` — the format the runbook's "Expected" blocks will quote).
3. **Run the failing exercise stub.** Same command, against the unsolved exercise. Capture what real failure output looks like (it's typically much noisier than first-pass intuition expects — see ex01's review-pass findings).
4. **Run empirical probes against the solution before drafting.** This is the highest-leverage step. Write a short throwaway program (in `src/inspect_<exNN>.clj` or `/tmp/inspect-<exNN>.clj`) that exercises each load-bearing claim you intend to make in the docs. Examples of what a probe should answer:
   - What does the chart look like as data? (`:id`, `:children`, `:node-type`)
   - What's in the configuration set at each lifecycle moment (before `start!`, after `start!`, after each event)?
   - What does the library's matching / selection / lookup function return for inputs at the edges (typos, missing keys, weird shapes)?
   - What happens when something the docstring says "shouldn't" happen happens anyway?

   The rule: **every empirical claim in the docs must be backed by output from a probe you actually ran.** If you're tempted to write "X probably returns Y" — write the probe instead. The ex01 first pass shipped multiple wrong claims because this step was skipped; the ex02 first pass was much closer to correct because the probes ran first.
5. **Skim the library namespaces the exercise touches.** Use the jar-inspection commands above. Particularly: the `com.fulcrologic.statecharts.elements` namespace for whichever element the exercise centers on (e.g., `parallel` for ex04, `invoke` for ex09).
6. **Re-read the top-level [`glossary.md`](./glossary.md).** If a term you're about to introduce in the module is actually cross-cutting, it belongs at the top level. If a top-level term turns out to need expansion specific to this exercise, the per-exercise glossary is the right place.
7. **Log open questions to [`ask_tony.md`](./ask_tony.md).** Some questions about the library can't be answered by reading source or running probes — they're about *intent* (is this behavior a bug or a feature?), *contract* (can I rely on this in future versions?), or *history* (why was it designed this way?). When you hit one of these, append it to `ask_tony.md` at the repo root rather than guessing in the docs. See the section below.

If any of these steps would be skipped for "this exercise is small," do them anyway. The smallest exercises in the source curriculum still produced wrong-content commits when verification was skipped.

---

## `ask_tony.md` — questions for the library author

Empirical verification answers most questions about what the library *does*. It can't answer questions about what the library *intends* — whether a behavior is a bug, a feature, or a coincidence; whether you can rely on it in future versions; what the historical reason for a design choice was.

When a doc would benefit from this kind of answer and you can't get it from source / probes / talks, append the question to [`ask_tony.md`](./ask_tony.md). Format:

```markdown
### Q: Short, specific question — one sentence.

**Where this came up.** ex0N's tutorial Step 3 — the docstring says X, but the code does X+Y. Need to know whether Y is intentional.

**What I verified empirically.** The behavior I observed, with the probe that produced it.

**What I can't tell from the code.** The intent / contract / history question that requires the author.

**Provisional answer in the docs.** What the curriculum currently says, which I'd update once the question is resolved.
```

Don't put speculation in the curriculum. If the docs need a claim and you can't verify it, either (a) cite the question's status as open and document the empirical behavior without claiming intent, or (b) defer the section until the question is answered. Inventing a rationale is worse than naming the gap.

The file is *append-only* during authoring; periodic cleanup (resolved questions removed, with answers folded back into the relevant docs) happens out of band.

---

## Audience baseline

When authoring, assume the reader:

- ✅ Can read Clojure — functions, maps, keywords, namespaces, threading macros, basic destructuring.
- ✅ Has installed Java + the Clojure CLI per [`setup.md`](./setup.md).
- ❌ Has never built a statechart and may have only the haziest notion of "state machine."
- ❌ Is *not* assumed to know Fulcro — these exercises don't require Fulcro app development knowledge, even though the library lives in the Fulcrologic ecosystem.

Concretely:

- **Do** introduce SCXML, "compound state," "region," "guard," "invocation" from zero, with a motivating example before the formal definition.
- **Do** explain Tony Kay's library-specific conventions when they first appear (`statechart` factory, `t/new-testing-env`, `op/assign`).
- **Don't** teach Clojure syntax. Don't explain what a keyword is, what `let` does, what `:refer` means.
- **Don't** assume Fulcro experience. If a chart pattern is Fulcro-flavored, call it out as such ("this is how the Fulcrologic library does it; the SCXML spec uses XML for the same idea").

---

## Anti-patterns to catch in review

These show up in first drafts. Catch them before commit.

### `;; =>` inline with executable code

Always two blocks: one for code, one for expected result. The learner who copy-pastes a block containing `;; =>` lines will paste the comments into the REPL and wonder why the REPL is confused.

### `<placeholder>` substitution

If the learner has to hand-edit `<event-keyword>` into `:next` before pasting, the runbook is not copy-paste-runnable. Bind it as a `def` once.

### Vague "see the source" pointers

❌ "Check the chart for a typo."
✅ "Open `src/exercises/ex03_guards.cljc:24` and confirm the guarded transition is *first* in document order."

### Solutions revealed before the learner has tried

If the runbook lays the full solution out before the exercise's tests have failed in front of the reader, the exercise loses its purpose. Use `<details><summary>▶ Show the answer</summary>…</details>` blocks for full-solution reveals, or frame them as "the target shape, for reference after you've tried it."

### Untested "Common breakages" entries

If you write "the symptom is X and the fix is Y," verify by deliberately breaking Y and confirming the symptom matches what you described. Theoretical breakages tend to drift from what learners actually experience.

### Quizzes that telegraph the answer

If the correct option is the only one written like a complete sentence and the distractors are five-word phrases, learners pattern-match. Make distractors substantive misconceptions, not filler.

### Tutorial sections without an exercise to ground them

If a tutorial concept has no chart element in the exercise that exercises it, ask whether the concept is actually load-bearing for this module. If it isn't, cut it. The exercises are the spine; tutorial content that doesn't map to an exercise is at risk of drifting from reality.

---

## Reviewing a module before commit

Read it once as a learner who has just completed the prior module. Then run through this checklist:

1. **Does the solution actually pass?** `clojure -M:test --focus exercises.ex0N-<topic>/<test-var>` from the exercise repo root — green?
2. **Tutorial → runbook → quiz → glossary order respected?** Each doc should be drawable from the one(s) before it.
3. **Per-exercise glossary doesn't redefine top-level terms?** Grep for top-level entries in the per-exercise file; replace with `[[term]]` references.
4. **No `;; =>` inside executable blocks?** Grep `;; =>`. Each occurrence should be in a result block, not a code block.
5. **No `<placeholder>` substitution?** Grep for `<` followed by a lowercase identifier. Replace with a `def`.
6. **Every "Try breaking it" ends with "Restore the X"?** Skim each entry.
7. **Quiz distractors are substantive?** Spot-check — no option should be obviously short relative to the others.
8. **The one-breath synthesis is actually one breath?** ~25–40 words. If it's three sentences, tighten.

If any item fails, fix it before commit.

---

## When in doubt

- **Can the learner paste this block into a fresh REPL without editing it?** If no, fix the runbook.
- **If this test fails, will the learner know which step failed and why?** If no, add a sanity check or a "Common breakages" entry.
- **Am I teaching a concept here?** If yes, that belongs in `tutorial.md` or `glossary.md`. The runbook *references* it; the runbook doesn't *re-derive* it.
- **Would a learner working alone on a Sunday afternoon be able to follow this without Slack-pinging a mentor?** That's the bar.

Runbooks aren't art. They're load-bearing infrastructure. The strongest praise a module can earn is "I followed this verbatim and the tests went green."
