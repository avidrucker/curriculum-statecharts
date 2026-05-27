# Curriculum — Statecharts (Tony Kay's `com.fulcrologic/statecharts`)

Curriculum companion to the [`statechart-exercises`](../statechart-exercises) repo — a hands-on tour through Tony Kay's Clojure(Script) statecharts library, aimed at getting a learner from "I read about statecharts once" to "I can read a chart, build one, and reason about why it transitions the way it does."

This is a **standalone educational repo**. It does not contain the exercises themselves (those live in [`statechart-exercises`](../statechart-exercises)) and does not contain the library code (that's [`com.fulcrologic/statecharts`](https://github.com/fulcrologic/statecharts)). It contains the **scaffolding** that turns the exercise set into a course: runbooks, glossaries, and quizzes per exercise, plus the top-level setup and orientation material.

---

## Intent

Three goals, balanced:

1. **High-quality learning.** Concepts are introduced in the order they're load-bearing for the next exercise, with vocabulary anchored to a 0–9 criticality scale and quizzes that probe real misconceptions (not filler distractors).
2. **Decent speed.** A learner working through the curriculum should be solving the first exercise within 30 minutes of cloning the exercise repo, and should finish all 11 exercises in roughly 10–20 hours of focused work — not three weeks.
3. **Maximally low stress.** Every command is copy-paste-runnable. Every test failure has a named diagnosis path. The implicit reader is someone who's *new to statecharts and possibly new to Fulcro*, working alone, with patience that erodes the moment they get stuck on tooling instead of concepts.

These three pull against each other (depth vs. speed, depth vs. low-stress). The curriculum's job is to keep them in tension without letting any one dominate.

---

## Source material

| Repo | Role |
| --- | --- |
| [`statechart-exercises`](../statechart-exercises) | **Subject.** Exercise stubs + solutions in `src/exercises/` and `src/solutions/`. 11 numbered Clojure(Script) files; each is a single chart + test suite. The set of solved exercises is the spine of every doc in this repo. |
| [`com.fulcrologic/statecharts`](https://github.com/fulcrologic/statecharts) | **Library.** Pinned to `1.2.25` in `statechart-exercises/deps.edn`. The library jar in `~/.m2` is the source of truth when curriculum docs disagree with API behavior. |
| [`fulcro-statecharts-talks`](../fulcro-statecharts-talks) | **Background.** Tony Kay's talk materials (Marathon + main talk). Linked from the top-level glossary entries for "statechart," "SCXML," and chart-level concepts. Not required reading, but the canonical lecture-track companion. |

Curriculum docs are downstream of these. **When a doc disagrees with the library source, the library wins.**

---

## Exercise map

| # | File | Topic | Concepts introduced |
| --- | --- | --- | --- |
| 1 | `ex01_compound_states.cljc` | Compound states & configuration | states, initial state, transitions, events |
| 2 | `ex02_event_matching.cljc` | Event matching | event name patterns, wildcards, hierarchy |
| 3 | `ex03_guards.cljc` | Guards (conditions) & document order | `:cond`, document order as fallback, session data |
| 3b | `ex03b_data_operations.cljc` | Data operations | `op/assign`, reading/writing session data |
| 3c | `ex03c_convenience.cljc` | Convenience macros | sugar over raw chart construction |
| 4 | `ex04_parallel.cljc` | Parallel states | regions running concurrently, event broadcast |
| 5 | `ex05_multi_target.cljc` | Multi-target transitions | one transition entering several states |
| 6 | `ex06_history.cljc` | History states | shallow vs. deep history, default targets |
| 7 | `ex07_internal_transitions.cljc` | Internal transitions | self-targeting without exit/entry |
| 8 | `ex08_final_and_done.cljc` | Final states & `done.state` | terminating a region, parent reaction |
| 9 | `ex09_invocations.cljc` | Child chart invocations | `invoke`, `:src`, `done.invoke` |

The numbering, including the `3b` / `3c` sub-letters, mirrors the exercise repo exactly so file-to-doc lookups stay one-to-one.

---

## Repo layout (target)

```
curriculum-statecharts/
├── overview.md                  this file — intent, strategy, doc-set definition
├── README.md                    quick-start: where to begin
├── CLAUDE.md                    authoring rules for content in this repo
├── setup.md                     how to clone, run tests, drive a REPL session
├── glossary.md                  top-level cross-cutting vocabulary (chart, region, event, ...)
├── statechart-concepts.md       optional: the conceptual primer that exercises assume
├── quiz-index.md                index of all per-exercise quizzes + scores tracker
│
├── ex01-compound-states/
│   ├── tutorial.md
│   ├── runbook.md
│   ├── glossary.md
│   └── quiz.md
├── ex02-event-matching/
│   ├── tutorial.md
│   ├── runbook.md
│   ├── glossary.md
│   └── quiz.md
├── ex03-guards/                 ── ex03, ex03b, ex03c each get their own module ──
├── ex03b-data-operations/
├── ex03c-convenience/
├── ex04-parallel/
├── ex05-multi-target/
├── ex06-history/
├── ex07-internal-transitions/
├── ex08-final-and-done/
└── ex09-invocations/
```

The directory naming uses dashes (`ex03-guards/`) rather than underscores (`ex03_guards/`) to mirror the curriculum-onboarding-rad-project convention; the *underlying file* in the exercise repo keeps its underscored Clojure-friendly name (`ex03_guards.cljc`).

---

## The four doc types

The source curriculum (`curriculum-onboarding-rad-project`) ships **five** primary docs per module: `exercises.md`, `tutorial.md`, `quiz.md`, `runbook.md`, `glossary.md`. This curriculum ships **four** — `tutorial.md`, `runbook.md`, `glossary.md`, `quiz.md` — dropping only `exercises.md` because the exercises already exist as code (stubs + worked solutions) in the sibling [`statechart-exercises`](../statechart-exercises) repo. Rewriting them in markdown would duplicate ground truth and risk drift.

The shape of each doc, adapted from the source curriculum:

### `tutorial.md` — the conceptual narrative ("why does this work this way?")

The *why* behind the runbook's *how*. Each tutorial:

- Opens with **"The story we're building"** — what the chart represents in the real world (a traffic light, a login gate, a checkout flow) and what behavior the tests will prove.
- Walks the concept in numbered steps that each map to one or more chart elements the runbook will then build. Steps go *concept first, then mechanics*: e.g., for ex03, the tutorial first establishes "what a guard is and why document order matters as a fallback," then shows the `:cond` syntax. The runbook then has the learner type it.
- Carries **`> Common misconception`** callouts inline — the source curriculum's convention for naming the wrong intuition explicitly before correcting it. (Real misconceptions sourced from places like "guards are tried in priority order — they're not, they're tried in document order.")
- Cross-references the [[glossary]] for high-criticality terms and the exercise file by name.
- Ends with a short **"What you should be able to explain"** list (3–5 bullets) — the conceptual bar the quiz will then test.

Target length: 200–500 lines per exercise; shorter for `ex01` (compound states are familiar to anyone who's used a state machine before), longer for `ex04` (parallel) and `ex09` (invocations).

### `runbook.md` — the "make it run and prove it" surface

For a unit-test-based exercise set (no long-running app, no browser, no `(reset-db!)` between steps), the runbook concept needs a slight rework. Instead of "which terminal, which command, which click target," the per-exercise runbook is:

1. **Opener** — what the learner will have built when the test suite goes green; estimated time.
2. **Read the exercise file first** — point at the docstring, the `TODO` comment, the test assertions. Name the 1–2 library namespaces the exercise relies on.
3. **REPL warm-up** — copy-paste a session that loads the namespace, evaluates the (still-stub) chart, and observes the initial test failure. Locks in the diagnostic loop.
4. **Build the chart, step by step** — numbered build steps. Each step adds one chart element (a state, a transition, a guard) and re-runs the relevant test, with the *expected* failure-becomes-pass output in a separate fenced block.
5. **Try this (REPL experiments)** — 2–4 self-contained probes that build the debugger's reflex: "When my chart doesn't transition, ask the REPL X." These are the operational equivalent of muscle memory.
6. **Try breaking it** — 1–3 deliberate edits to the *finished* chart (swap document order, remove a guard, drop a transition target) with predicted failures and an explicit "**Restore the X**" at the end of each.
7. **Common breakages** — symptom-indexed: "the test fails with `No matching transition`," "the test fails with `Should this be a .cljs file?`," etc. Each entry walks the bisection.
8. **What success looks like** — a 5–8 item checklist plus a one-breath synthesis (~25–40 words) the learner narrates back.

Target length: 200–400 lines per exercise; longer for `ex04` (parallel), `ex09` (invocations).

The four load-bearing conventions from the source curriculum carry over verbatim: terminal labels on every code block, reusable `def`s instead of `<placeholder>` substitution, executable code separated from expected results, and symptom-indexed diagnostics.

### `glossary.md` — vocabulary with criticality ratings

Plain-language definitions first; cross-references second. Each entry is rated on the **0–9 criticality scale** (9 = you cannot move past this exercise without it; 0 = trivia worth knowing).

Per-exercise glossaries are *additive* to the top-level `glossary.md` — they cover terms first introduced in that exercise (`:cond` and `op/assign` first appear in ex03; "parallel region" first appears in ex04). Cross-cutting terms ("event," "state," "transition") live in the top-level glossary and are referenced from per-exercise glossaries via `[[chart]]`-style links.

Target length: 8–20 entries per exercise.

### `quiz.md` — recall-level multiple choice

12–16 MCQs per exercise (the source curriculum's range), one correct answer per question, distractors that reflect real misconceptions — not filler. Answer key at the bottom as a `# | Answer | Quick reason` table. Quizzes draw from the runbook's build steps and the glossary's high-criticality entries.

---

## Authoring order

Adapted from `curriculum-onboarding-rad-project/CLAUDE.md`. The order is **not** alphabetical and **not** runbook-first — it's the order that minimizes the risk of inventing API surface that doesn't exist:

1. **Verify against the exercise + solution files.** Read the exercise stub and the solution side by side. Run the solution against the test suite (`clj -X:test` filtered to that exercise). Confirm the test assertions pass with the solution. *This is the ground truth for every downstream doc.*
2. **Draft `tutorial.md`** — the conceptual narrative behind the solved exercise. The tutorial answers "why does this work this way?" before the runbook tells the learner what to type.
3. **Draft `runbook.md`** — the operational pass. Now you know what the learner must type and in what order, and the tutorial has established the *why* the runbook can lean on.
4. **Draft `quiz.md`** — questions probe the concepts the tutorial established and the runbook exercised; distractors reflect realistic misconceptions.
5. **Draft `glossary.md`** — last, because by now you've surfaced every term the other three docs use, and ranking each on the 0–9 criticality scale is informed by which terms turned out to be load-bearing.

### Audience baseline (mandatory mental model when authoring)

Assume the reader **can read Clojure** (functions, maps, keywords, namespaces, threading macros) but is **new to statecharts** as a concept *and* **new to Tony Kay's library specifically**. They may also be new to Fulcro — but the exercises don't require Fulcro app knowledge, so Fulcro orientation stays light (a single footnote in `setup.md` pointing at the `fulcro-book` and `fulcro-statecharts-talks` repos for context).

Concretely:

- ✅ Do introduce SCXML, "compound state," "region," "guard," "invocation" from zero, with motivating examples before formal definitions.
- ✅ Do explain Tony Kay's library-specific conventions (`statechart` factory, `t/new-testing-env`, `op/assign`) when they first appear.
- ❌ Don't teach Clojure syntax (don't explain what a keyword is, what `let` does, etc.).
- ❌ Don't assume Fulcro experience. If a chart concept has a Fulcro-flavored idiom, call it out as "this is how Fulcro statecharts does it; the SCXML spec is similar but uses XML."

---

## Status

- [x] `overview.md` — this file.
- [ ] Top-level `setup.md`, `glossary.md`, `quiz-index.md`, `CLAUDE.md`.
- [ ] Per-exercise module sets (11 modules × 3 docs = 33 files).
- [ ] Optional secondary pass (questions + answers per module), if the user wants it after the primary pass lands.

A reasonable first slice is the top-level setup + the ex01 module set; once that pattern is approved, ex02–ex09 follow the same shape.
