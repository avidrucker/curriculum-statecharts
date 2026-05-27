# Runbook — ex06: History States (Shallow & Deep)

The operational pass. By the end you'll have edited `src/exercises/ex06_history.cljc` to define a settings-panel chart with deep history and seen all nine assertions go green.

**Estimated time:** 40–60 minutes if you've completed ex01–ex05. The chart is structurally the densest of the curriculum so far (three nesting levels, history element, transitions on parent for ancestor-walk).

---

## Prerequisites

- You've completed ex01–ex05. Comfortable with nested compound states, the ancestor walk for transitions (ex02), and `:initial` on a state.
- You've read [`tutorial.md`](./tutorial.md) for ex06. The runbook references "history node," "default target," "shallow vs deep," "ancestor walk" without re-defining them.

---

## Where commands run

Same setup as previous exercises.

| Label | What it runs | How to start |
| --- | --- | --- |
| **REPL** | `clojure -M:nrepl` | from the exercise repo root |
| **Shell** | `clojure -M:test` | from the exercise repo root |

---

## Step 1 — Read the exercise file first

Open `src/exercises/ex06_history.cljc`. ~82 lines. Four things to notice:

1. **The docstring** sketches the chart's structure as a tree (`:app > :settings > {:general, :privacy > {:privacy-basic, :privacy-advanced}, :notifications}` + `:main-screen` sibling of `:settings`). Three levels of nesting; one history node.
2. **The TODO comment** tells you (a) put the history node inside `:settings`, (b) target the history node's ID (not `:settings`) from `:open-settings`, (c) put tab-switching transitions on `:settings` for the ancestor-walk pattern.
3. **The Hint:** `Use :initial on :app to start in :main-screen (not :settings)` — overriding document order is necessary because `:settings` would otherwise be the initial.
4. **The `deftest`** has seven `testing` blocks containing **9 `is` assertions** total. The longest block ("re-open restores deep history") has 2 `is` calls.

---

## Step 2 — Run the test and see it fail

**Shell, exercise repo root.** Start watch mode:

```sh
clojure -M:test --focus exercises.ex06-history/history-test --watch
```

Summary line:

```
1 tests, 9 assertions, 9 failures.
```

Keep the watch terminal open.

---

## Step 3 — Add the elements require

The exercise stub doesn't import the elements. You need a new one — `history`:

```clojure
  (:require
    [clojure.test :refer [deftest is testing]]
    [com.fulcrologic.statecharts.chart :refer [statechart]]
    [com.fulcrologic.statecharts.elements :refer [history state transition]]
    [com.fulcrologic.statecharts.testing :as t]))
```

Save. Still `9 failures`.

---

## Step 4 — Build the chart, layer by layer

Five edits. The chart's interconnectedness (3 nesting levels, transitions referencing children that don't yet exist) means we go skeleton-first, then fill in tabs and transitions.

### 4.1 — Skeleton: `:app` with `:main-screen` initial + `:settings` placeholder

Replace `;; YOUR CODE HERE`:

```clojure
(def settings-panel
  (statechart {}
    (state {:id :app :initial :main-screen}
      (state {:id :settings})
      (state {:id :main-screen}))))
```

Save.

Expected — **2 of 9 pass**:

```
1 tests, 9 assertions, 7 failures.
```

What passes:

- Block 1's `(t/in? env :main-screen)` — `:initial :main-screen` on `:app` overrides document order so the chart lands at `:main-screen`.
- Block 4's `(t/in? env :main-screen)` *after `:close-settings`* — the chart never left `:main-screen` (no `:open-settings` transition exists yet, so the earlier `:open-settings` was silently dropped). It's still in `:main-screen` when block 4 checks.

The "passes by accident" rule strikes again — assertions that check the start state pass for free when no transition has fired yet.

### 4.2 — Add the three tabs + `:privacy`'s sub-states

```clojure
(state {:id :settings}
  (state {:id :general})
  (state {:id :privacy}
    (state {:id :privacy-basic})
    (state {:id :privacy-advanced}))
  (state {:id :notifications}))
```

(Leave the rest of the chart unchanged.)

Save.

Expected — **still 2 of 9 pass**:

```
1 tests, 9 assertions, 7 failures.
```

We added structure but no transitions, so no events do anything. The chart still starts in `:main-screen` and never moves; the same two assertions pass.

### 4.3 — Add `:open-settings` and `:close-settings` (no history yet)

Two transitions on `:app`:

```clojure
(state {:id :app :initial :main-screen}
  (state {:id :settings} ...)
  (state {:id :main-screen})
  (transition {:event :open-settings :target :settings})         ; targets compound, not history
  (transition {:event :close-settings :target :main-screen}))
```

Save.

Expected — **4 of 9 pass**:

```
1 tests, 9 assertions, 5 failures.
```

Trace:

- Block 1's `(t/in? env :main-screen)` ✓ — initial state.
- Block 2's `(t/in? env :general)` after `:open-settings` ✓ — `:open-settings` enters `:settings` (first doc-order child = `:general`).
- Block 3's `:tab-privacy` has no handler → chart stays in `:general` → both block-3 assertions fail.
- Block 4's `(t/in? env :main-screen)` after `:close-settings` ✓ — clean return.
- Block 5's two assertions (`:privacy-advanced`, `:privacy`) both fail — re-open lands at `:general` (default), not the restored deep history.
- Block 6's `:tab-notifications` has no handler → chart stays in `:general` → fail.
- Block 7's `:tab-general` has no handler → chart stays in `:general` → **passes by accident** (`(t/in? env :general)` is true because we never left).

The block-7 "pass by accident" is the kind of false positive that makes test-driven authoring tricky. The transition isn't wired; the chart happens to *be* in `:general` for unrelated reasons.

### 4.4 — Add the tab-switching transitions and `:privacy-detail`

Put `:tab-general`, `:tab-privacy`, `:tab-notifications` on `:settings` (ancestor-walk pattern). Put `:privacy-detail` on `:privacy-basic`:

```clojure
(state {:id :settings}
  (transition {:event :tab-general :target :general})
  (transition {:event :tab-privacy :target :privacy})
  (transition {:event :tab-notifications :target :notifications})
  (state {:id :general})
  (state {:id :privacy}
    (state {:id :privacy-basic}
      (transition {:event :privacy-detail :target :privacy-advanced}))
    (state {:id :privacy-advanced}))
  (state {:id :notifications}))
```

Save.

Expected — **7 of 9 pass**:

```
1 tests, 9 assertions, 2 failures.
```

Tabs now work. Block 3 fully passes (`:tab-privacy` lands at `:privacy-basic`; `:privacy-detail` lands at `:privacy-advanced`). Block 6 and block 7 also pass (`:tab-notifications`, `:tab-general`). Block 5's *re-open* assertions still fail: targeting `:settings` directly re-enters at `:general` (the first child), not at `:privacy-advanced`.

### 4.5 — Add the deep history node + re-target `:open-settings`

Two edits in one step: add the `history` element inside `:settings`, and change `:open-settings`'s target from `:settings` to the history node's ID.

```clojure
(state {:id :settings}
  (history {:id :settings-history :type :deep} :general)         ; ← new
  (transition {:event :tab-general :target :general})
  (transition {:event :tab-privacy :target :privacy})
  (transition {:event :tab-notifications :target :notifications})
  (state {:id :general})
  (state {:id :privacy}
    (state {:id :privacy-basic}
      (transition {:event :privacy-detail :target :privacy-advanced}))
    (state {:id :privacy-advanced}))
  (state {:id :notifications}))

;; ... and at the :app level:
(transition {:event :open-settings :target :settings-history})   ; ← changed from :settings
```

Save.

Expected — **all 9 pass**:

```
1 tests, 9 assertions, 0 failures.
```

What block 5 now does:

- `:open-settings` targets `:settings-history`.
- The history node was recorded at exit (block 4's `:close-settings`), with `:privacy-advanced` as the deep-recorded state.
- Re-entry restores `:privacy-advanced` — exactly what block 5 asserts[^p2].

You've solved ex06.

---

## Try this (REPL experiments)

**REPL, exercise repo root.** Connect and load:

```clojure
(require '[exercises.ex06-history :as ex06] :reload)
(require '[com.fulcrologic.statecharts.testing :as t])
```

### Probe 1: What does the chart contain after a full close + re-open cycle?

```clojure
(defn config [env]
  (-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
      :storage deref :test
      (get :com.fulcrologic.statecharts/configuration)))

(def env (t/new-testing-env {:statechart ex06/settings-panel
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(config env)
;; => #{:app :main-screen}

(t/run-events! env :open-settings)
(config env)
;; => #{:general :settings :app}  (first visit, default target = :general)

(t/run-events! env :tab-privacy :privacy-detail)
(config env)
;; => #{:privacy :settings :app :privacy-advanced}

(t/run-events! env :close-settings)
(config env)
;; => #{:app :main-screen}    (history records :privacy-advanced as last)

(t/run-events! env :open-settings)
(config env)
;; => #{:privacy :settings :app :privacy-advanced}    (deep history restores!)
```

Expected[^p2]: empirically verified. The first `:open-settings` lands at `:general` (default); the second lands at `:privacy-advanced` (history restoration).

**Why this probe exists.** Watching the configuration shift across a full close+re-open cycle is the cleanest way to internalize "history actually works." The default-then-restore pattern is the load-bearing intuition.

### Probe 2: Shallow vs deep — the visible difference

```clojure
(require '[com.fulcrologic.statecharts.chart :refer [statechart]])
(require '[com.fulcrologic.statecharts.elements :refer [history state transition]])

(def chart-shallow
  (statechart {}
    (state {:id :app :initial :main-screen}
      (state {:id :settings}
        (history {:id :sh :type :shallow} :general)
        (transition {:event :tab-privacy :target :privacy})
        (state {:id :general})
        (state {:id :privacy}
          (state {:id :privacy-basic}
            (transition {:event :privacy-detail :target :privacy-advanced}))
          (state {:id :privacy-advanced})))
      (state {:id :main-screen})
      (transition {:event :open-settings :target :sh})
      (transition {:event :close-settings :target :main-screen}))))

(def env2 (t/new-testing-env {:statechart chart-shallow
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env2)
(t/run-events! env2 :open-settings :tab-privacy :privacy-detail)
(t/in? env2 :privacy-advanced)              ; => true
(t/run-events! env2 :close-settings :open-settings)
(t/in? env2 :privacy)                       ; => true   (shallow restored :privacy)
(t/in? env2 :privacy-basic)                 ; => true   (privacy's default initial)
(t/in? env2 :privacy-advanced)              ; => false  (sub-state lost)
```

Expected[^p3]: shallow restores `:privacy` (the immediate child of `:settings`) but loses the deep state — `:privacy` then re-enters at its own default (`:privacy-basic`). Compare to deep history (Probe 1), where `:privacy-advanced` itself is restored.

**Why this probe exists.** Seeing the difference makes the deep/shallow choice concrete. For UIs with nested navigation, deep is almost always what you want; for flat tab-only navigation, shallow is sufficient.

### Probe 3: Targeting the compound directly skips history

```clojure
(def chart-no-history
  (statechart {}
    (state {:id :app :initial :main-screen}
      (state {:id :settings}
        (history {:id :sh :type :deep} :general)
        (transition {:event :tab-privacy :target :privacy})
        (state {:id :general})
        (state {:id :privacy}
          (state {:id :privacy-basic}
            (transition {:event :privacy-detail :target :privacy-advanced}))
          (state {:id :privacy-advanced})))
      (state {:id :main-screen})
      ;; HERE: target :settings directly, NOT :sh
      (transition {:event :open-settings :target :settings})
      (transition {:event :close-settings :target :main-screen}))))

(def env3 (t/new-testing-env {:statechart chart-no-history
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env3)
(t/run-events! env3 :open-settings :tab-privacy :privacy-detail)
(t/in? env3 :privacy-advanced)              ; => true
(t/run-events! env3 :close-settings :open-settings)
(t/in? env3 :privacy-advanced)              ; => false  (default-initial again)
(t/in? env3 :general)                       ; => true
```

Expected[^p4]: the history node is *defined* in the chart but never consulted, because the transition targets `:settings` directly. The chart enters `:settings`'s default initial (`:general`) on every `:open-settings`.

**Why this probe exists.** Knowing that a history node only works when *targeted by ID* prevents the common bug of "I added history, why isn't it remembering?" The answer is usually "your transition targets the compound, not the history node."

### Probe 4: When does history record?

```clojure
(def env4 (t/new-testing-env {:statechart ex06/settings-panel
                               :mocking-options {:run-unmocked? true}} {}))
(t/start! env4)
(t/run-events! env4 :open-settings :tab-privacy :privacy-detail)
(t/in? env4 :privacy-advanced)              ; => true

;; Now go to :general WITHOUT leaving :settings
(t/run-events! env4 :tab-general)
(t/in? env4 :general)                       ; => true

;; Now close + re-open
(t/run-events! env4 :close-settings :open-settings)
(t/in? env4 :general)                       ; => true   ← :general was the last visited
(t/in? env4 :privacy-advanced)              ; => false
```

Expected[^p5]: history records what was active *at the moment of exit*. The earlier visit to `:privacy-advanced` is overwritten by the subsequent within-compound transition to `:general`. Only the state immediately before `:close-settings` matters.

**Why this probe exists.** "History snapshots on every transition" is a tempting wrong model. The real behavior — "history snapshots on exit" — matches SCXML semantics and is more efficient. Watching the override happen makes the rule stick.

---

## Try breaking it

### Break 1: Target the compound state directly

**Edit.** Change `:open-settings`'s `:target` from `:settings-history` to `:settings`:

```clojure
(transition {:event :open-settings :target :settings})
```

**Predicted symptom.** Block 5's `(t/in? env :privacy-advanced)` fails — re-opening lands at `:general` (the default initial of `:settings`) instead of restoring history[^p4]. Blocks 1–4 still pass; the regression is only in block 5.

**What this proves.** Targeting the compound directly skips history *even when the history node is defined*. History is opt-in via the target ID, not automatic.

**Restore `:settings-history`** before continuing.

### Break 2: Use shallow history instead of deep

**Edit.** Change `:type :deep` to `:type :shallow`:

```clojure
(history {:id :settings-history :type :shallow} :general)
```

**Predicted symptom.** Block 5's `(t/in? env :privacy-advanced)` fails — shallow history restores `:privacy` but lands at `:privacy-basic` (privacy's default initial)[^p3]. The block's second assertion `(t/in? env :privacy)` still passes (because shallow did restore the right tab).

**What this proves.** Shallow vs deep matters when the restored state has its own nested children. For top-level tab navigation (no nesting), shallow is enough; for nested navigation, deep preserves the exact sub-state.

**Restore `:type :deep`** before continuing.

### Break 3: Drop the default target from `history`

**Edit.** Remove the third argument:

```clojure
(history {:id :settings-history :type :deep})        ; no default target
```

**Predicted symptom.** Chart construction throws: `Wrong number of args (1) passed to: com.fulcrologic.statecharts.elements/history`[^p6]. The chart never loads; the test runner reports the error.

**What this proves.** The default target is required by the `history` function's signature. The library asserts on it; there's no "fallback to compound's initial" behavior.

**Restore `:general`** as the default target before continuing.

---

## Common breakages

### `Wrong number of args (1) passed to: history`

You called `(history {...})` without the default-target third argument. Add `:general` (or whichever state you want as the first-visit fallback).

### `Cannot register invalid chart`

A `:target` references a state ID that doesn't exist. Most commonly: the transition targets `:settings-history` but the chart's history node has a different `:id` (or no history node defined yet). Check the IDs match.

### Block 5's re-open lands at `:general`, not `:privacy-advanced`

The transition's target is `:settings` (the compound), not `:settings-history` (the history node). See Break 1.

### Block 5 lands at `:privacy-basic`, not `:privacy-advanced`

The history node is `:type :shallow`. Change to `:type :deep` (see Break 2).

### `:tab-privacy` works from `:general` but not from `:notifications`

The transition is declared on `:general` (or some specific tab), not on `:settings`. Move it to `:settings` so the ancestor walk picks it up from any tab.

### Chart starts on the settings panel instead of `:main-screen`

You forgot `:initial :main-screen` on `:app`. Document order picks the first child, which is `:settings` (declared first in the solution). Add the `:initial` keyword to override.

---

## What success looks like

A checklist:

- [ ] `src/exercises/ex06_history.cljc` requires `statecharts.elements` and refers `history`, `state`, `transition`.
- [ ] `:app` has `:initial :main-screen` (overrides document order).
- [ ] `:settings` contains a `(history {:id :settings-history :type :deep} :general)` element.
- [ ] `:open-settings` targets `:settings-history` (not `:settings`).
- [ ] `:tab-general`, `:tab-privacy`, `:tab-notifications` are declared on `:settings` itself (ancestor walk).
- [ ] `clojure -M:test --focus exercises.ex06-history/history-test` reports `9 assertions, 0 failures.`.
- [ ] You can predict what happens if you swap `:deep` to `:shallow` (Break 2's symptom).
- [ ] You ran Probe 4 (history records on exit) and saw `:general` win over `:privacy-advanced` because it was visited last.

**Explain in one breath:**

> *"`:open-settings` targets the history node `:settings-history`, not the compound `:settings`. First visit uses the history's default target `:general`. After visiting `:privacy-advanced` and closing, history records `:privacy-advanced` at exit. Re-opening via the history node restores `:privacy-advanced` exactly — because `:type :deep` captures the full descent chain."*

If that's natural, ex06 is internalized. Move on to [`quiz.md`](./quiz.md), then [`gotchas.md`](./gotchas.md), then ex07 (internal transitions).

---

> **Verified empirically (probes run during ex06 authoring):**
>
> - [^p2]: Full solution end-to-end — deep history correctly restores `:privacy-advanced` after close + re-open
> - [^p3]: Shallow history restores `:privacy` (the immediate child) but lands at `:privacy-basic` (default initial)
> - [^p4]: Targeting `:settings` directly (skipping history) lands at `:general` (default initial)
> - [^p5]: History snapshots at exit time; within-compound transitions update what's recorded
> - [^p6]: `(history {...})` without default-target throws `Wrong number of args (1)` at construction
