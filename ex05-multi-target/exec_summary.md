# Executive Summary — ex05: Multi-Target Transitions

The 5-minute refresher. Use this when you've completed ex05 once and want to re-anchor.

---

## The one big idea

**A transition's `:target` accepts a vector of state IDs, not just a single ID.** When the transition fires, all named states become active simultaneously. The use case: precisely controlling which leaf becomes active in each region of a parallel.

The honest secondary lesson: **for this exercise's specific test, multi-target isn't strictly necessary** — single-target works because the unspecified region defaults to its document-order initial, which happens to match what the test expects. Multi-target is for *non-default* initials.

---

## What we built

ex04's media player, plus a `:reset` transition that returns the chart to `:stopped` AND `:unmuted` regardless of where it was.

```clojure
(def media-player-with-reset
  (statechart {}
    (parallel {:id :player}
      (transition {:event :reset :target [:stopped :unmuted]})         ; ← new, multi-target
      (state {:id :playback}
        (state {:id :stopped}  (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}
          (transition {:event :play :target :playing})
          (transition {:event :stop :target :stopped})))
      (state {:id :volume}
        (state {:id :unmuted} (transition {:event :toggle-mute :target :muted}))
        (state {:id :muted}   (transition {:event :toggle-mute :target :unmuted}))))))
```

One transition added; behavior identical to ex04 except for the new `:reset` event.

---

## Mechanics in 30 seconds

- **`:target [:a :b]`** = enter both `:a` AND `:b` simultaneously. Each target should be in a different region of the parallel; the library doesn't enforce this, and same-region targets produce a structurally-invalid configuration with both atomic states active in one region (see gotcha #2).
- **Single-keyword `:target :stopped`** is normalized to a one-element vector internally. The library's `transition` element does this in `elements.cljc`. So `:target :stopped` ≡ `:target [:stopped]`.
- **Order in the vector doesn't matter.** `[:stopped :unmuted]` and `[:unmuted :stopped]` produce identical configurations.
- **Single-target re-entry uses default initials in unspecified regions.** If a transition's target is one region's leaf, the OTHER region falls back to its document-order initial (or `:initial`). For ex05's test, `:unmuted` IS volume's default — so `:target :stopped` would ALSO pass the test.
- **Multi-target is for precision, not for "this chart has multiple regions."** Use it when the default initial isn't what you want.

---

## Composes with

- **ex04 (parallel)** provides the regions multi-target precisely populates.
- **ex06 (history)** is the complement: history *restores* whatever was active before exit; multi-target *prescribes* exactly which leaves to enter.

---

## Gotchas to remember

- **Same-region multi-target is silently accepted.** `:target [:stopped :playing]` (both in `:playback`) leaves the chart with BOTH atomic states active in one region — structurally invalid. The library doesn't validate region-uniqueness. The chart self-corrects on the next event that exits one of the over-active states, but during the broken window, `t/in?` returns true for both. Assertions can pass for the wrong reason.
- **Multi-target on a non-parallel chart produces the same kind of invalid state.** If `:a` and `:b` are sibling top-level states (not parallel regions), `:target [:a :b]` puts BOTH in the configuration, violating the "one active child" rule at the chart root.
- **Misspelled targets throw `Cannot register invalid chart`** at registration time — same opaque error as ex02 gotcha #7.
- **Multi-target doesn't replace default-initial entry, it OVERRIDES it.** For regions you don't name in the vector, the runtime falls back to defaults.
- **Vector with one element ≡ single keyword** — the library doesn't distinguish them. Pick the form that signals intent.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex05-multi-target :as ex05])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex05/media-player-with-reset
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/run-events! env :play :toggle-mute)
(t/in? env :playing)                  ; → true
(t/in? env :muted)                    ; → true

(t/run-events! env :reset)
(t/in? env :stopped)                  ; → true   (multi-target effect)
(t/in? env :unmuted)                  ; → true   (multi-target effect)
```

If those four checks pass, ex05 is solid. Move on to [ex06 (history states)](../ex06-history/) — the complement: instead of prescribing exact leaves on entry, history *remembers* and *restores* whatever was last active.
