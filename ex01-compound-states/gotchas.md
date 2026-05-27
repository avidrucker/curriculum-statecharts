# Gotchas — ex01: Compound States & Configuration

Silent failures, hidden behaviors, and easy-to-miss configurations that bite when nothing looks wrong. See [`gotchas-overview.md`](../gotchas-overview.md) for what belongs here vs. in `runbook.md`'s "Common breakages."

Each entry verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. An empty `(statechart {})` builds fine and runs fine — and does nothing useful

**What you'll see.** Your exercise stub starts as `(def traffic-light (statechart {} ;; YOUR CODE HERE))`. Loading the namespace, building a testing env, and calling `(t/start! env)` *all succeed*. No exception. The first assertion just returns `false` because there's no `:red` state.

**What's really happening.** The `statechart` factory at `chart.cljc:73` validates only that *top-level children have allowed `:node-type`s*. An empty children vector passes that check trivially. The chart is built with `:id :ROOT`, `:children []`, and the runtime is happy to "run" it — the configuration after `start!` is `#{}` (empty set).

**Defense.** When `(t/in? env :something)` returns `false` for every state name and you're sure you started the chart, check whether the chart actually has any states. `(:children traffic-light)` should be non-empty (at minimum it'll contain the auto-`:initial` plus your declared states).

*See also:* [tutorial Step 2](./tutorial.md), [glossary `chart root (:ROOT)`](./glossary.md).

---

### 2. Forgetting `:id` on a state silently auto-assigns one

**What you'll see.** You write `(state {} (transition {:event :go :target :other}))` (oops, no `:id`). The chart compiles. `(t/start! env)` succeeds. Then `(t/in? env :the-name-you-intended)` returns false.

**What's really happening.** `elements.cljc`'s `new-element` (line 60) does `(merge {:id (or id (genid (name type)))} attrs ...)` — a missing `:id` gets a generated keyword like `:state25839`. The chart works internally (the generated ID is real and consistent), but you can't reference it from `:target` keywords or `t/in?` checks because you never knew its name. On ClojureScript builds the library warns; on the JVM there's *no* warning.

**Defense.** Always set `:id` explicitly on every `state` and `transition`. If you `pprint` a chart and see a `:state-NNNNN` or `:initial-NNNNN` you didn't write, that's an auto-assigned ID — confirm it's not on a state you intended to name.

---

### 3. `(t/in? env :ROOT)` returns `false`, even while the chart is running

**What you'll see.** You inspect the chart with `(:id ex01/traffic-light)` and get `:ROOT` — the chart root is clearly named `:ROOT`. You then check `(t/in? env :ROOT)` after `start!` and get `false`. The root *exists* but isn't "in" the configuration.

**What's really happening.** The auto-generated chart root is the chart's *container* — every chart has one — but the runtime doesn't put `:ROOT` in the `::sc/configuration` set. The set holds active atomic states and (in nested charts) active compound-state ancestors below `:ROOT`. The root is conceptually "always active" but excluded from the set.

**Defense.** Don't ever check `(t/in? env :ROOT)` expecting it to mean "is the chart running?" Use `(seq (-> env :env (get :com.fulcrologic.statecharts/working-memory-store) :storage deref :test (get :com.fulcrologic.statecharts/configuration)))` for that, or just check for a known atomic state. The ex01 [tutorial Step 5](./tutorial.md) table is the canonical answer for what's in the set at each lifecycle moment.

**Important caveat (introduced in ex03):** the rule above is for charts driven via `start!` + `run-events!`. The test-only helper `t/goto-configuration!` (used starting in ex03) builds its target configuration via a different algorithm that *does* include `:ROOT`. So `(t/in? env :ROOT)` can return `true` after a `goto-configuration!` call. See [ex03 gotchas #1](../ex03-guards/gotchas.md) for the asymmetry.

---

### 4. `(t/in? env :anything)` returns `false` silently — typos don't error

**What you'll see.** You meant `:red` but typed `:readd`. `(t/in? env :readd)` returns `false`. No "no such state" warning. Your test fails for what looks like the right reason ("chart isn't in `:readd`") but the actual problem is the typo.

**What's really happening.** `testing.cljc:271` shows `t/in?` is implemented as `(contains? configuration-set state-name)`. `contains?` doesn't care whether the key is meaningful — anything not in the set returns `false`. The function has no view of the chart's declared state IDs.

**Defense.** When a `t/in?` check unexpectedly fails, copy the state name out of your `(state {:id ...})` declaration into the check, don't retype it. Use editor multi-cursor or the kill-yank-by-keyword trick. If you can `pprint` the chart and see your intended `:id`, the issue is downstream of declaration.

---

### 5. Unknown events are silently dropped

**What you'll see.** You wrote `:event :nextt` (extra t) in a transition, then `(t/run-events! env :next)` in your test. The chart doesn't move. No error. The next `(t/in? env :green)` check fails — and you have no log line saying "I didn't find a matching transition for `:next`."

**What's really happening.** Per SCXML, a chart that has nothing to do with an event is supposed to drop it silently — this is a *feature* (charts are robust to spurious events). The library follows this. There is debug-level logging in `v20150901-impl.cljc` that records the matching outcome, but it's not surfaced by default.

**Defense.** When a `run-events!` call seems to have no effect, copy the transition's `:event` keyword and the test's event keyword and `=`-compare them in the REPL. Or use `(events/name-match? <candidate> <event>)` directly to confirm the matcher returns true. See ex01 [runbook Probe 3](./runbook.md).

---

### 6. `t/start!` *can* be called twice — and the second call resets the chart

**What you'll see.** You start a chart, run some events, want to test a different scenario. You call `(t/start! env)` again expecting an error or no-op. Instead the chart silently returns to its initial configuration.

**What's really happening.** `testing.cljc:246`'s `start!` calls the underlying processor's start and saves new working memory — overwriting whatever was there. It doesn't check whether the env was already running. Calling it twice is effectively a reset.

**Defense.** This works for cases where you want to reset, but the **fresh-env pattern** is clearer and matches the convention of ex02 (which creates `env2` and `env3` for independent scenarios):

```clojure
(def env (t/new-testing-env {:statechart traffic-light
                              :mocking-options {:run-unmocked? true}}
                            {}))
(t/start! env)
;; ... run scenario 1

(def env2 (t/new-testing-env {:statechart traffic-light
                               :mocking-options {:run-unmocked? true}}
                             {}))
(t/start! env2)
;; ... run scenario 2 independently
```

Anyone reading your test should be able to see that scenario 2 starts from scratch; with a `start!` reset they have to know about the reset behavior.

---

### 7. The chart options `{}` first argument is required *positionally*, not by name

**What you'll see.** You write `(statechart (state {:id :red}))`, forgetting the options map. The chart compiles. No error. But the chart's root has weird attrs and no children.

**What's really happening.** `statechart`'s signature is `[attrs & children]`. When you skip the options map, your first state becomes `attrs` — the chart's *attribute map*. The factory then forces `:id :ROOT` and `:node-type :statechart` on it, but the leftover attrs (like `:children` from the state) get stomped or ignored. The resulting chart has no real states.

**Defense.** Always write the options map, even if empty: `(statechart {} ...)`. If you `pprint` the chart and the root has unexpected keys that look like state attributes, this is the bite.

---

### 8. Re-evaluating `(def chart ...)` in the REPL doesn't update an existing testing env

**What you'll see.** You edit the chart, re-evaluate the form in your REPL, then run your tests. The tests fail in the way they did *before* your edit. You wonder why the edit didn't take.

**What's really happening.** A testing env holds a reference to the chart definition it was constructed with. Re-defining the var `traffic-light` doesn't reach into existing envs and update them — they still point at the old chart value. (Clojure's `def` swaps the var; live references to the old value don't follow.)

**Defense.** After re-evaluating a chart `def`, create a *fresh* testing env with `t/new-testing-env`. The exercise's `deftest` block does this every run, so the test runner is immune; the bite is when you're poking at things interactively at the REPL.

```clojure
;; in REPL
(require '[exercises.ex01-compound-states :as ex01] :reload)
(def env (t/new-testing-env {:statechart ex01/traffic-light
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
;; ... fresh env always
```

---

### 9. The `elements` require is *not* in the exercise stub by default

**What you'll see.** You type `(state {:id :red} ...)` in the exercise file and the editor flags `state` as an unresolved symbol. Or the test errors with `Unable to resolve symbol: state`.

**What's really happening.** Each `ex0N_*.cljc` stub only requires what the *to-be-written* test code strictly needs (`chart`, `testing`). The `elements` namespace, which exports `state`, `transition`, `parallel`, `final`, `history`, `invoke`, etc., is *not* required — it's the learner's job to add it.

**Defense.** First move in any exercise: open the file, add `[com.fulcrologic.statecharts.elements :refer [state transition]]` to the `(:require ...)` form (plus whatever other elements the exercise needs). This is in [runbook Step 3](./runbook.md) but worth keeping in muscle memory across the whole curriculum.

---

### 10. `clj` vs `clojure` on Linux: one needs `rlwrap`, the other doesn't

**What you'll see.** You run `clj -M:test` and get `Please install rlwrap for command editing or use "clojure" instead.` The command fails before kaocha even starts.

**What's really happening.** The Clojure CLI ships two scripts. `clj` wraps `clojure` in `rlwrap` for readline history at the REPL prompt. If `rlwrap` isn't installed (default on some Linux distros, common on minimal installs), `clj` errors. `clojure` works the same way without the wrapping.

**Defense.** Use `clojure` instead of `clj` in your shell scripts and habits. Or `sudo apt install rlwrap` (Mint/Ubuntu) once. `setup.md` covers this — but it's an easy paper-cut to hit on a fresh machine.

*Not strictly a library gotcha, but it bites within the first 5 minutes of trying to run anything.*
