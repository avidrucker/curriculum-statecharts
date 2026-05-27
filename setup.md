# Setup

Getting the exercise repo running on your machine and into a state where you can solve one exercise, run its tests, and see green output.

**Time budget:** 10–20 minutes if your JDK and Clojure CLI are already installed; 30–60 minutes if you're installing the JDK or Clojure for the first time.

---

## Prerequisites

| What | Why | How to check |
| --- | --- | --- |
| Java 21+ | Clojure runs on the JVM | `java -version` |
| Clojure CLI (`clojure` / `clj`) | Runs the deps + test runner | `clojure --version` |
| `git` | Cloning the exercise repo | `git --version` |

If you're missing any of these:

- **Java 21:** install Eclipse Temurin or Amazon Corretto. On Linux Mint / Ubuntu, `sudo apt install openjdk-21-jdk` works. On macOS, `brew install --cask temurin@21`.
- **Clojure CLI:** [https://clojure.org/guides/install_clojure](https://clojure.org/guides/install_clojure) — pick the script for your OS.

You do **not** need: shadow-cljs, Node, npm, Datomic, a database, or a browser. The exercises are pure Clojure unit tests.

---

## Repo layout

The curriculum and the exercises live in **two sibling directories**. The curriculum assumes both are checked out side-by-side:

```
~/Documents/Study/Fulcro/
├── statechart-exercises/        ← the exercises (subject material)
└── curriculum-statecharts/      ← this repo (companion docs)
```

You're reading the companion. If you haven't cloned the exercise repo yet:

```sh
cd ~/Documents/Study/Fulcro
git clone <statechart-exercises-url> statechart-exercises
```

(Replace `<statechart-exercises-url>` with the actual remote — ask your mentor / check your Slack / `gh repo list` if you have GitHub CLI configured.)

---

## A note on `clj` vs `clojure`

The Clojure CLI ships two scripts:

- **`clojure`** — runs Clojure directly. No external dependencies.
- **`clj`** — wraps `clojure` in `rlwrap` for editable history at the REPL prompt. Requires `rlwrap` to be installed.

If `clj` errors with `Please install rlwrap for command editing or use "clojure" instead`, either install `rlwrap` (`sudo apt install rlwrap` on Mint/Ubuntu, `brew install rlwrap` on macOS) or substitute `clojure` for `clj` in every command below. They take the same arguments.

This document uses `clojure` for compatibility. Most editor / IDE integrations also call `clojure`, not `clj`.

## First run — warm the deps cache

The first time you enter the exercise repo, deps need to be downloaded. This takes a minute and only needs to happen once per dep version:

```sh
cd ~/Documents/Study/Fulcro/statechart-exercises
clojure -P -M:test
```

Expected: a flurry of `Downloading…` lines followed by a clean prompt. No errors.

> **Why `-P -M:test` and not `-A:test -P`?** The `-A` form with a `:main-opts`-bearing alias triggers a deprecation warning (`WARNING: Use of :main-opts with -A is deprecated. Use -M instead.`). The `-P` flag asks the CLI to download deps and exit without executing the main. `-M:test` activates the `:test` alias for deps resolution.

If you see `Could not find artifact com.fulcrologic/statecharts:1.2.25`, your Maven mirror is misconfigured — most often a stale `~/.m2/settings.xml` or a corporate proxy. Resolve by checking `~/.m2/settings.xml` for the right mirror URLs.

---

## Run the test runner

The exercise repo uses **kaocha** as the test runner, wired through the `:test` alias in `deps.edn`. The kaocha config in `tests.edn` defines two test suites: `:exercises` (stubs you'll fill in) and `:solutions` (the worked answers).

### Run everything

```sh
clojure -M:test
```

> **Note on `-M` vs `-X`.** Some of the exercise file docstrings say `clj -X:test`. That's a stale instruction — the alias uses `:main-opts ["-m" "kaocha.runner"]`, which requires `-M`, not `-X`. (`-X:test` would error: `No :exec-fn`.) Use `-M:test` (with either `clojure` or `clj`).

Expected on a clean exercise checkout (no exercises solved yet): the `:exercises` suite reports failing tests, the `:solutions` suite reports green. The fact that solutions pass is your sanity check that the toolchain works.

### Run a single suite

```sh
clojure -M:test --focus :exercises
clojure -M:test --focus :solutions
```

### Run one exercise's tests

To re-run just one exercise's tests while you're working on it:

```sh
clojure -M:test --focus exercises.ex01-compound-states/traffic-light-test
```

The argument is `<namespace>/<test-var>`. Both come from the exercise file's `(ns …)` and `(deftest …)` forms — the namespace uses dashes (not underscores).

### Watch mode (re-run on file change)

```sh
clojure -M:test --watch
```

Combine with `--focus` to limit re-runs to the file you're working on. Stop with `Ctrl-C`.

---

## REPL workflow (recommended)

Running `clojure -M:test` per change works, but a REPL is faster once you're iterating. The exercise repo has an `:nrepl` alias for editor-attached REPLs:

```sh
clojure -M:nrepl
```

Expected output (one line):

```
nREPL server started on port 41359 on host localhost - nrepl://localhost:41359
```

The port number varies each run. The number is also written to `.nrepl-port` in the exercise repo's root — most editor plugins read this file automatically.

### From the REPL: load an exercise + run its tests

Once you're connected (whether via editor or via `clojure -M:nrepl` and then a separate `clj` client):

```clojure
(require '[exercises.ex01-compound-states :as ex01] :reload)
(clojure.test/run-tests 'exercises.ex01-compound-states)
```

The `:reload` flag picks up edits to the file since the namespace was first loaded. Without it, the REPL keeps the old definitions and you'll wonder why your fix didn't take.

For a tighter loop while you're filling in a chart:

```clojure
(in-ns 'exercises.ex01-compound-states)
;; edit the file, save it
(require '[exercises.ex01-compound-states] :reload)
(clojure.test/run-tests)
```

### Inspect a chart manually

The testing namespace gives you a "testing env" you can drive interactively:

```clojure
(require '[com.fulcrologic.statecharts.testing :as t])
(def env (t/new-testing-env {:statechart ex01/traffic-light
                             :mocking-options {:run-unmocked? true}}
                            {}))
(t/start! env)
(t/in? env :red)         ; what state is the chart in?
(t/run-events! env :next)
(t/in? env :green)       ; cycle and re-check
```

Each runbook's "Try this (REPL experiments)" section leans on this pattern. Get comfortable with it now.

---

## Editor integration (optional)

Any editor that speaks nREPL will work. Common pairings:

| Editor | Plugin | Connects via |
| --- | --- | --- |
| IntelliJ + Cursive | Cursive (commercial / free for non-commercial) | nREPL — point at `.nrepl-port` |
| VS Code | Calva | "Connect to a running REPL" → reads `.nrepl-port` |
| Emacs | CIDER | `M-x cider-connect-clj` → reads `.nrepl-port` |
| Vim / Neovim | Conjure or vim-fireplace | `:CocCommand`s vary |

None are required. If you're new to Clojure tooling, start with `clojure -M:test --watch` in one terminal and your editor of choice for editing; add REPL integration once the test-driven loop feels routine.

---

## Common breakages

Symptom-indexed. If you hit something here, the diagnosis lives below the symptom.

### `Could not locate exercises/ex01_compound_states__init.class`

You're requiring the namespace with dashes (`exercises.ex01-compound-states`) — that's correct. The file on disk has underscores (`ex01_compound_states.cljc`) — that's also correct; it's the Clojure convention. The error means the file is not on the classpath.

Check that you're in the exercise repo root (`pwd` should end in `statechart-exercises`) when you start `clj`. The `deps.edn` `:paths ["src"]` only resolves relative to the working directory.

### `No such namespace: com.fulcrologic.statecharts.chart`

The statecharts library jar didn't make it into your classpath. Run `clojure -P -M:test` again — the `-P` flag downloads dependencies without running anything.

### Kaocha hangs at "Starting…"

Some Linux distros have an old Babashka or a slow `java` on first JIT. Give it 30 seconds. If still hung past 60s, kill it with `Ctrl-C` and re-run with `-J-Dclojure.main.report=stderr` to surface JVM-level errors:

```sh
clojure -J-Dclojure.main.report=stderr -M:test
```

### Tests pass for `:solutions` but fail for `:exercises` — and you haven't started yet

That's expected. The `:exercises` suite is the stub set you'll be filling in. Solutions passing confirms the library is loading correctly.

### `Should this be a .cljs file?`

You're trying to run ClojureScript-only code in a JVM REPL. None of these exercises require ClojureScript — every file ends in `.cljc` (works in both), but the test runner runs the JVM side. If you see this, you've added a require for something ClojureScript-only (e.g., `goog.string`) that doesn't belong.

---

## What "done with setup" looks like

A checklist:

- [ ] `clojure --version` prints `1.12.x` or newer.
- [ ] `java -version` prints `21.x` or newer.
- [ ] `clojure -P -M:test` completes with no errors.
- [ ] `clojure -M:test --focus :solutions` runs and reports all green (the solutions are pre-baked).
- [ ] `clojure -M:test --focus :exercises` runs and reports failures (the stubs are empty — that's the point).
- [ ] You can open a REPL with `clojure -M:nrepl`, type `(+ 1 1)`, and see `2` back.

When all six are checked, you're set up. Open [`overview.md`](./overview.md) to confirm the curriculum's intent, then start with [`ex01-compound-states/tutorial.md`](./ex01-compound-states/tutorial.md).
