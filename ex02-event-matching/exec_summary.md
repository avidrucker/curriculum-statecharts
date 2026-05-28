# Executive Summary — ex02: Hierarchical Event Names & Prefix Matching

The 5-minute refresher. Use this when you've completed ex02 once and want to re-anchor without re-reading `tutorial.md`.

---

## The one big idea

**Event names are dot-separated segments. Transition matching is a *segment-prefix* check. And when an event arrives, the runtime walks the active state's ancestors looking for the first matching transition.**

These three rules compose into "specific transitions in deep states + catch-all transitions in ancestors" — the canonical way to express "handle this specifically, or fall back to a general handler."

---

## What we built

An error handler. Specific events (`:error.critical`, `:error.recoverable`) get dedicated transitions inside the `:working` state; everything else (`:error.foo`, `:error.unknown`, etc.) is caught by a single `:error → :error-display` transition on `:operational` (the parent compound).

```clojure
(def error-handler
  (statechart {}
    (state {:id :operational}
      (state {:id :working}
        (transition {:event :error.critical    :target :shutdown})
        (transition {:event :error.recoverable :target :recovery}))
      (state {:id :recovery})
      (state {:id :error-display})
      (transition {:event :error :target :error-display}))         ; catch-all
    (state {:id :shutdown})))
```

Three new things vs. ex01: dotted event names, a nested compound state, and a transition on a parent state that catches what its children don't.

---

## Mechanics in 30 seconds

- **Event names are segments.** `:error.critical` is `["error" "critical"]`. Hyphens stay inside a segment; only dots split.
- **Prefix matching:** `:event :error` matches `:error`, `:error.x`, `:error.x.y` — anything starting with `["error"]`. Wildcard `:error.*` is equivalent.
- **Ancestor walk:** when no transition in the active state matches, the runtime checks the parent, then grandparent, etc. **First match wins.**
- **A failed guard counts as "no match"** — the ancestor walk continues. (Important for ex03.)
- **Nested compound states have multi-element configurations.** When in `:working`, the configuration is `#{:operational :working}` — both parent and active child.
- **`name-match?` is namespace-strict.** `:my/error` does NOT match `:error`. Either both have the same namespace or neither has one.

---

## The big footgun

**`name-match?` has an undocumented string-prefix fallback for non-namespaced candidates.** For non-namespaced events, after the segment-prefix check fails, the matcher falls back to plain `(str/starts-with? event-name candidate)`. So:

- `:error` matches `:errors` (string `":errors"` starts with `":error"`)
- `:error` matches `:error-thing`
- `:error.crit` matches `:error.critical`

**This footgun bites in real charts later in the curriculum** — ex06 (history target collisions), ex08 (`:done.state.X` collisions), ex09 (`:done.invoke.X` collisions). It's the curriculum's most cross-cutting bug pattern.

Defense: never name two events such that one is a string-prefix of another.

---

## Composes with

- **ex01** established the chart-is-data and configuration-is-set foundations.
- **ex03** introduces guards (`:cond`) — the ancestor walk continues past *failed* guards, not just unmatched events.
- **ex04** introduces `parallel` — event broadcast across regions is essentially the ancestor walk applied independently per region.
- **ex06, ex08, ex09** all hit the string-prefix matching footgun in real chart scenarios.

---

## Gotchas to remember

- **String-prefix fallback** — see "the big footgun" above.
- **Where the catch-all transition lives matters.** Placing `:error → :error-display` inside `:working` (alongside the specific transitions) would still work, but document order alone would resolve which fires — fragile. Placing on `:operational` (parent) makes "specifics win" structural, not order-dependent.
- **Transitions at the chart root are rejected.** Wrap them in a state.
- **Reorder of non-overlapping transitions doesn't matter.** Order only matters when multiple transitions could match the same event (ex03 makes this load-bearing).
- **`:operational` is in the config when `:working` is** — compound parents are also "in." `(t/in? env :operational)` and `(t/in? env :working)` both return `true` simultaneously.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex02-event-matching :as ex02])
(require '[com.fulcrologic.statecharts.testing :as t])
(require '[com.fulcrologic.statecharts.events :as evt])

(def env (t/new-testing-env {:statechart ex02/error-handler
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/in? env :working)              ; → true
(t/in? env :operational)          ; → true   (compound parent is also in)

(t/run-events! env :error.critical)
(t/in? env :shutdown)             ; → true   (specific match wins)

;; The footgun:
(evt/name-match? :error :errors)  ; → true   ← surprising
```

If those four checks behave as shown, ex02 is solid. Move on to [ex03 (guards)](../ex03-guards/) for conditional transitions and the data model.
