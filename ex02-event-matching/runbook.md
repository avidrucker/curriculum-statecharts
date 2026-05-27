# Runbook — ex02: Hierarchical Event Names & Prefix Matching

The operational pass. By the end of this document you'll have edited `src/exercises/ex02_event_matching.cljc` to define a working error-handler chart and seen all four assertions go green.

**Estimated time:** 20–35 minutes if you've completed ex01; the chart is structurally bigger than ex01's traffic light and there's a new concept (nested compound state) to type for the first time.

---

## Prerequisites

- You've completed ex01 — your `setup.md` checklist is green and you've watched a chart's tests go from red to green at least once.
- You've read [`tutorial.md`](./tutorial.md) for ex02. The runbook references its terms (prefix matching, ancestor walk, configuration set) without re-defining them.

The cross-cutting [`glossary.md`](../glossary.md) covers anything from ex01 you've forgotten; the per-exercise [`glossary.md`](./glossary.md) covers ex02-specific vocabulary.

---

## Where commands run

Same setup as ex01:

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` — interactive REPL | from the exercise repo root |
| **Shell** | `clojure -M:test` — the kaocha test runner | from the exercise repo root |

All paths below are relative to `statechart-exercises/` (the sibling of this curriculum repo).

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex02_event_matching.cljc` in your editor. ~65 lines. Three pieces to notice:

1. **The docstring** describes the target structure as a tree (`:operational > :working`, with `:error.critical`, `:error.recoverable`, `:error` transitions placed at specific levels). Don't skim past this — the *placement* of each transition is the whole point of the exercise.
2. **The `(ns …)` form** requires `statecharts.chart` and `statecharts.testing` — but **not** `statecharts.elements`. As in ex01, you'll add it yourself.
3. **The `deftest`** is the spec. Four assertions, but two of them create *fresh testing envs* (`env2`, `env3`) so each scenario starts cleanly:
   - Assertion 1: starts in `:working`.
   - Assertion 2: `:error.critical` lands in `:shutdown` (using `env`).
   - Assertion 3: `:error.recoverable` lands in `:recovery` (using fresh `env2`).
   - Assertion 4: `:error.unknown` lands in `:error-display` (using fresh `env3`).

The fresh-env pattern is here because once the chart transitions to `:shutdown` (a leaf with no outgoing transitions), it's stuck — you can't drive the same env back to `:working` to test another scenario. Each `let` block sets up an independent run. You'll see this pattern again whenever a chart has a one-way path.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start a watch-mode runner so you see results as you save:

```sh
clojure -M:test --focus exercises.ex02-event-matching/event-matching-test --watch
```

The output, as in ex01, is noisy — every failing assertion dumps the full ~2500-char env map and DEBUG logging is interleaved. Skip to the summary line at the bottom:

```
1 tests, 4 assertions, 4 failures.
```

Keep this terminal open. Every save to the exercise file will trigger a re-run.

---

## Step 3 — Add the elements require

Same first move as ex01. The exercise stub doesn't import `state` or `transition`. Open `src/exercises/ex02_event_matching.cljc` and add the elements require to the `(ns …)` form:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. The watch-mode runner re-runs and still reports `4 failures` — that's expected.

---

## Step 4 — Build the chart, one layer at a time

You'll do this in four edits. The pattern is: add structure, observe which assertions pass, identify which is next to satisfy.

### 4.1 — The compound `:operational` and its initial child `:working`

Replace `;; YOUR CODE HERE` with the compound state + its initial child:

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}))))
```

Save. Expected — **first assertion passes; the other three fail**:

```
1 tests, 4 assertions, 3 failures.
```

Why? The chart now has a top-level `:operational` compound state with `:working` as its first (and only) child. `(t/start! env)` enters `:operational`, then enters `:working` (initial-by-document-order). The configuration becomes `#{:operational :working}`. `(t/in? env :working)` returns `true` — the first assertion is satisfied. The remaining three fail because there are no error transitions and no `:shutdown`, `:recovery`, or `:error-display` states yet.

This is the moment compound-state nesting first matters: a single state declaration created **two** configuration entries.

### 4.2 — Add the specific `:error.critical` handler and `:shutdown`

The second assertion (`:error.critical` → `:shutdown`) needs:

1. A transition inside `:working` that listens for `:error.critical` and targets `:shutdown`.
2. A `:shutdown` state at the top level (a sibling of `:operational`).

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical :target :shutdown})))
    (state {:id :shutdown})))
```

Save. Expected — **two assertions pass**:

```
1 tests, 4 assertions, 2 failures.
```

Why? When `:error.critical` arrives, the ancestor walk starts at `:working`. `:working` has a matching transition. The chart exits `:working`, exits `:operational`, enters `:shutdown`. The configuration goes from `#{:operational :working}` to `#{:shutdown}`. `(t/in? env :shutdown)` is now `true`.

### 4.3 — Add `:error.recoverable` and `:recovery`

The third assertion needs the same pattern, one level over:

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical :target :shutdown})
        (transition {:event :error.recoverable :target :recovery}))
      (state {:id :recovery}))
    (state {:id :shutdown})))
```

Save. Expected:

```
1 tests, 4 assertions, 1 failures.
```

Why? `:working` now has two transitions; one matches `:error.recoverable` and targets `:recovery`. `:recovery` is a sibling of `:working` inside `:operational` (not a top-level state). After the transition, the configuration becomes `#{:operational :recovery}` — `:operational` is still active because the move was *within* it.

### 4.4 — Add the catch-all and `:error-display`

The last assertion: `:error.unknown` should be caught by the parent's general handler. This needs:

1. A transition on `:operational` (not on `:working`) listening for `:error` and targeting `:error-display`.
2. A `:error-display` state that's a sibling of `:working` and `:recovery`.

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical :target :shutdown})
        (transition {:event :error.recoverable :target :recovery}))
      (state {:id :recovery})
      (state {:id :error-display})
      (transition {:event :error :target :error-display}))
    (state {:id :shutdown})))
```

Save. Expected — **all four assertions pass**:

```
1 tests, 4 assertions, 0 failures.
```

Why? When `:error.unknown` arrives, `:working` has no match (its transitions are `:error.critical` and `:error.recoverable`, both fail the prefix check against `:error.unknown`). The ancestor walk falls through to `:operational`. `:operational`'s `:error` transition matches by prefix — `:error` is a segment-prefix of `:error.unknown`. The chart enters `:error-display`. Configuration: `#{:operational :error-display}`.

You've solved ex02.

---

## Try this (REPL experiments)

Each probe builds a debugger's reflex you'll lean on in later exercises.

**REPL, exercise repo root.** Connect (`clojure -M:nrepl` then your editor, or plain `clojure`), then:

```clojure
(require '[exercises.ex02-event-matching :as ex02] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
(require '[com.fulcrologic.statecharts.events :as evt])
```

### Probe 1: What's in the configuration at each stage?

**Scenario.** You want a clear picture of how the configuration set grows and shrinks as the chart moves.

```clojure
(def env (t/new-testing-env
           {:statechart ex02/error-handler
            :mocking-options {:run-unmocked? true}}
           {}))
(t/start! env)
(t/in? env :operational)        ; → true
(t/in? env :working)            ; → true
(t/in? env :ROOT)               ; → false (always)

(t/run-events! env :error.unknown)
(t/in? env :operational)        ; → true  (still active; just moved within)
(t/in? env :error-display)      ; → true
(t/in? env :working)            ; → false (exited)
```

Expected: when the chart is in `:working`, both `:operational` and `:working` are "in." After moving to `:error-display`, `:operational` is still in (because `:error-display` is one of its children) but `:working` is not.

**Why this probe exists.** The ancestor-in-configuration behavior is foreign if you've only used flat state machines before. Every subsequent exercise leans on it. If you can reach for `(t/in? env :operational)` to check "are we anywhere in the operational region?", debugging nested charts becomes much easier.

### Probe 2: Test `name-match?` directly

**Scenario.** You're writing a chart with a transition listening for `:user.*` and want to confirm what it actually catches.

```clojure
(evt/name-match? :error :error.critical)   ; → true
(evt/name-match? :error :error.unknown)    ; → true
(evt/name-match? :error :warning)          ; → false
(evt/name-match? :error :error.foo.bar)    ; → true (deep prefix)
(evt/name-match? :error.* :error.x)        ; → true (wildcard stripped)
```

Expected: prefix-on-segments works as the tutorial describes. `:warning` shares no first segment with `:error`, so no match.

**Why this probe exists.** When a transition isn't firing and you can't tell whether the matcher is the problem, call `name-match?` directly. It's the same function the runtime uses; if it returns `false` for your inputs, your transition won't fire.

### Probe 3: The string-prefix footgun

**Scenario.** You want to verify the surprise behavior described in tutorial Step 6.

```clojure
(evt/name-match? :error :errors)         ; → true  (!)
(evt/name-match? :error :error-thing)    ; → true  (!)
(evt/name-match? :error.crit :error.critical)  ; → true  (!)
(evt/name-match? :error :erroneous)      ; → false (':erroneous' doesn't start with ':error')
```

Expected: the first three return `true` even though they're not segment-clean prefixes. The library's `name-match?` falls back to raw string prefix matching for non-namespaced candidates.

**Why this probe exists.** If you ever wonder why a transition listening for `:user` is catching a `:users.created` event, this is why. The defense is **don't name events such that one is a string-prefix of another** unless you intend them to be aliased.

### Probe 4: Namespaces are strict

**Scenario.** You're integrating with a Fulcro app that uses namespaced events and want to confirm namespace matching.

```clojure
(evt/name-match? :my/error :my/error)         ; → true
(evt/name-match? :my/error :my/error.x)       ; → true
(evt/name-match? :my/error :other/error)      ; → false (namespaces differ)
(evt/name-match? :error :my/error)            ; → false (candidate has no ns, event does)
```

Expected: with namespaces involved, the matcher requires exact namespace equality.

**Why this probe exists.** Namespaced events are how Fulcro-flavored apps avoid name collisions across subsystems. Knowing the matcher is namespace-strict means you can confidently use `:auth/error` and `:billing/error` as fully independent event names.

---

## Try breaking it

Three deliberate edits to your finished chart. Each builds intuition for a rule the tutorial described.

### Break 1: Put a transition at the chart root

**Edit.** Move the catch-all transition out of `:operational` and place it as a direct sibling of `:operational` and `:shutdown`:

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical :target :shutdown})
        (transition {:event :error.recoverable :target :recovery}))
      (state {:id :recovery})
      (state {:id :error-display}))
    (transition {:event :error :target :error-display})   ; ← at chart root
    (state {:id :shutdown})))
```

**Predicted symptom.** The chart factory throws at load time:

```
Illegal top-level node. Root node cannot have: #{:transition} elements.
```

Your editor's REPL or `clojure -M:test` will surface the exception before the test even runs.

**What this proves.** Transitions only live inside states (or other elements that legally contain them, like `parallel` and `final`). The chart root is restricted to `:state :parallel :final :data-model :script` — you can verify this in the library source at `com/fulcrologic/statecharts/chart.cljc` (the `legal-node-types` set).

To get a "global" catch-all in practice, you wrap the whole chart in one outer `state` and put the transition there.

**Restore the original chart** before continuing.

### Break 2: Reorder `:error.critical` and `:error.recoverable`

**Edit.** Swap the two transitions inside `:working`:

```clojure
(state {:id :working}
  (transition {:event :error.recoverable :target :recovery})    ; ← now first
  (transition {:event :error.critical :target :shutdown}))
```

**Predicted symptom.** All tests still pass. The two transitions listen for *different* events, so document order doesn't matter — `:error.critical` is the only match for `:error.critical`, regardless of which transition is declared first.

**What this proves.** Document order only matters when *multiple transitions could fire on the same event* (relevant in ex03, with guards). When event names don't overlap, the ordering is cosmetic.

**Restore the original order** before continuing.

### Break 3: Move the catch-all into `:working`

**Edit.** Move the `:error → :error-display` transition *inside* `:working`, alongside the specific handlers:

```clojure
(state {:id :working}
  (transition {:event :error.critical :target :shutdown})
  (transition {:event :error.recoverable :target :recovery})
  (transition {:event :error :target :error-display}))    ; ← moved in
```

**Predicted symptom.** All assertions still pass… for the wrong reason. Now `:working` handles every error event, including `:error.critical` and `:error.recoverable` — but document order makes `:error.critical` match first (it's declared above `:error`), so the behavior is unchanged for the test events.

**Try this to surface the bug:** add a fifth assertion (just in the REPL, not the file):

```clojure
(def env (t/new-testing-env {:statechart ex02/error-handler
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env :error.recoverable)
(t/in? env :recovery)             ; → still true
;; BUT — does document order save us? In what order are the matches checked?
```

Then *swap* the order so the catch-all is declared first:

```clojure
(state {:id :working}
  (transition {:event :error :target :error-display})   ; ← FIRST now
  (transition {:event :error.critical :target :shutdown})
  (transition {:event :error.recoverable :target :recovery}))
```

Re-run the tests. **Now assertions 2 and 3 fail** — because `:error.critical` is matched by the *general* `:error` transition first (document order), shipping the chart to `:error-display` instead of `:shutdown`.

**What this proves.** The "more specific handlers take priority" rule is really *placement-by-state + document-order within a state*. If you collapse all handlers into one state, you lose the placement axis and document order alone has to carry the disambiguation — and it does so by *first match*, not by *specificity*.

**Restore the original chart** before continuing.

---

## Common breakages

Symptom-indexed.

### `Illegal top-level node. Root node cannot have: #{:transition} elements.`

You placed a `transition` element directly at the chart root, alongside states. The chart root only accepts `:state :parallel :final :data-model :script`. Wrap your top-level transition inside a `state` element (or move it inside an existing state).

### `Cannot register invalid chart`

One of your `:target` keywords references a state ID that doesn't exist anywhere in the chart. The chart's validator runs at registration time (inside `t/new-testing-env`) and throws this generic error — **it does not name which target is bad or which transition contains it**. The error is opaque on purpose, not a curriculum oversight; verify upstream by filing a clarification (see [`ask_tony.md`](../ask_tony.md)).

To diagnose: grep your chart definition for `:target` and confirm every target keyword is also declared somewhere as `:id`:

```sh
grep -oE ':target [^ )]+' src/exercises/ex02_event_matching.cljc
grep -oE ':id [^ }]+' src/exercises/ex02_event_matching.cljc
```

Diff the two lists. Anything in the first that isn't in the second is the typo. (See also [gotchas #7](./gotchas.md).)

### Assertion 1 fails: `starts in :working` returns false

Three things to check:

1. **Is `:operational` declared as a state and `:working` as its child?** If you wrote `(state {:id :working} ...)` as a sibling of `:operational` rather than a child, `:working` is a top-level state and the chart will pick `:operational` *or* `:working` as the initial — whichever comes first in document order. Open the file and confirm `:working` is *inside* the `:operational` form.
2. **Is `:operational` first in document order?** If `:shutdown` is declared before `:operational`, the chart starts in `:shutdown`. Move `:operational` to be the first top-level state.
3. **Did you forget `(t/start! env)`?** Unlikely if you're using the provided test, but worth a check.

### Assertion 2 fails: `:error.critical` doesn't reach `:shutdown`

The chart probably routed `:error.critical` to `:error-display` instead — meaning the catch-all `:error` transition fired first. This happens when:

- The catch-all is placed *inside* `:working` (not on `:operational`) and is declared *before* the specific transitions, OR
- The catch-all is placed in `:working` *after* the specifics, but you've also accidentally added a catch-all to the chart root that fires before the ancestor walk reaches `:working`'s specifics.

Open the file. The structure must be:

```
:operational
  :working                  ← the SPECIFICS live here
    :error.critical → :shutdown
    :error.recoverable → :recovery
  :error → :error-display   ← the CATCH-ALL lives at this level, not deeper
```

### Assertion 4 fails: `:error.unknown` doesn't reach `:error-display`

Two checks:

1. **Is `:error-display` actually a state?** The transition's `:target` resolves to a state's `:id`; if the state doesn't exist, the transition silently doesn't fire (or fails the validator depending on the library version).
2. **Is the `:error` transition inside `:operational` (or higher), not inside `:working`?** When the chart is in `:working` and `:error.unknown` arrives, `:working` has no match. The ancestor walk goes to `:operational`. If the `:error` transition isn't on `:operational`, the walk continues to `:ROOT`, which also has nothing — and the event is dropped.

### Tests pass but `:error.recoverable` ends up in `:shutdown`

You wrote `:error.critical` twice (or `:error.recoverable` twice) by accident. Open the file; confirm each transition has a distinct `:event` keyword.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex02_event_matching.cljc` requires `statecharts.elements` and refers `state` and `transition`.
- [ ] `error-handler` is defined with the structure described in Step 4.4 (top-level `:operational` compound + `:shutdown` sibling).
- [ ] `clojure -M:test --focus exercises.ex02-event-matching/event-matching-test` reports `4 assertions, 0 failures.`.
- [ ] You can predict, without running the chart, where `:error.network` will end up. (Spoiler: `:error-display`. Because: `:working` has no match for `:error.network`; the ancestor walk reaches `:operational`'s `:error` transition; that prefix-matches.)
- [ ] You can predict the configuration immediately after the chart starts. (`#{:operational :working}`.)
- [ ] You ran Probes 1, 2, 3, and 4 and saw the empirical results match the tutorial.

**Explain in one breath:**

> *"`:error.critical` matches the specific handler in `:working` and routes to `:shutdown`. `:error.unknown` doesn't match `:working`'s transitions, so the ancestor walk falls through to `:operational`'s catch-all `:error` transition, which prefix-matches and routes to `:error-display`. Document order and state depth together determine which transition wins."*

If that's natural, ex02 is internalized. Move on to [`quiz.md`](./quiz.md), then ex03 (guards).
