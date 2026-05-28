# curriculum-statecharts

Curriculum companion to the sibling [`statechart-exercises`](../statechart-exercises) repo — an 11-exercise tour through Tony Kay's `com.fulcrologic/statecharts` library, aimed at getting a learner from "I read about statecharts once" to "I can read a chart, build one, and reason about why it transitions the way it does."

Each exercise in the upstream repo gets its own directory here with six docs:

- `tutorial.md` — conceptual narrative
- `runbook.md` — operational pass; build the chart step by step
- `quiz.md` — multiple-choice recall checks
- `glossary.md` — per-module vocabulary
- `gotchas.md` — silent failures and hidden behaviors that bite when nothing looks wrong
- `exec_summary.md` — 5-minute refresher for returning to a module weeks later

## Layout

```
overview.md             intent + strategy (read this first if you're new)
setup.md                cloning, test runner, REPL workflow
concepts-map.md         visual concept map (Mermaid diagrams; reading guide by experience level)
glossary.md             top-level cross-cutting vocabulary
gotchas-overview.md     guidance for authoring per-exercise gotchas.md files
ask_tony.md             open questions for the library author (intent / contract / history)
issues.md               curriculum-wide tracker of plausible-but-unverified claims
CLAUDE.md               authoring rules for content in this repo

ex01-compound-states/   per-exercise module (6 docs each)
ex02-event-matching/
ex03-guards/            ── ex03, ex03b, ex03c are separate modules ──
ex03b-data-operations/
ex03c-convenience/
ex04-parallel/
ex05-multi-target/
ex06-history/
ex07-internal-transitions/
ex08-final-and-done/
ex09-invocations/
```

## Where to start

- **First time here?** Read [`overview.md`](./overview.md) — explains intent, the doc types, and the audience.
- **Want a visual roadmap before diving in?** [`concepts-map.md`](./concepts-map.md) has six Mermaid diagrams from "where to start" to "the full picture."
- **Setting up the project for the first time:** [`setup.md`](./setup.md) walks the JDK + Clojure CLI + first test run.
- **Working through an exercise:** open that exercise's directory and read `tutorial.md` first, then work through `runbook.md` with the exercise file open in your editor.
- **Returning to a module weeks later?** Each module has an `exec_summary.md` — a 5-minute refresher that re-anchors the core ideas without re-reading the tutorial.
- **Authoring or revising content:** [`CLAUDE.md`](./CLAUDE.md) is the source of truth for conventions; [`overview.md`](./overview.md) covers the *why*.
- **Looking up a term:** the top-level [`glossary.md`](./glossary.md) covers cross-cutting vocabulary; per-exercise glossaries cover terms specific to that module.

## Status

| Module | Tutorial | Runbook | Glossary | Quiz | Gotchas | Exec Summary |
| --- | :---: | :---: | :---: | :---: | :---: | :---: |
| ex01-compound-states | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex02-event-matching | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex03-guards | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex03b-data-operations | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex03c-convenience | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex04-parallel | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex05-multi-target | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex06-history | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex07-internal-transitions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex08-final-and-done | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ex09-invocations | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Top-level scaffolding (`overview.md`, `setup.md`, `concepts-map.md`, `glossary.md`, `CLAUDE.md`) is in place. **All 11 per-exercise modules are complete** with their full 6-doc set (tutorial, runbook, quiz, glossary, gotchas, exec_summary).

## Upstream

The curriculum is downstream of:

- **[`../statechart-exercises`](../statechart-exercises)** — the exercises themselves (stubs + worked solutions).
- **[`com.fulcrologic/statecharts`](https://github.com/fulcrologic/statecharts)** — the library, pinned to `1.2.25` in the exercise repo's `deps.edn`.
- **[`../fulcro-statecharts-talks`](../fulcro-statecharts-talks)** — Tony Kay's talk materials (lecture-track companion).

When this repo's docs disagree with the library source, the source wins — see [`CLAUDE.md`](./CLAUDE.md)'s "Source-of-truth verification order" section.
