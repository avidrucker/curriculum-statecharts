# Executive Summary — ex04: Parallel Regions

The 5-minute refresher. Use this when you've completed ex04 once and want to re-anchor.

ex04 is the first **dense** module — and where the configuration set finally contains multiple atomic states active simultaneously.

---

## The one big idea

**`parallel` activates ALL its children simultaneously, not just one.** Each child is a "region" — its own compound state, evolving independently. Events broadcast to every region; each region's matching transitions fire.

Three consequences:

- The configuration is a set with *multiple atomic leaves* (one per region) plus their compound ancestors.
- Region independence is *structural*, not declared — it emerges from each region only declaring transitions for events it cares about.
- A transition with a target *outside* the parallel exits the WHOLE parallel (all regions at once) per SCXML's LCA rule.

---

## What we built

A media player with two independent concerns: playback (`:stopped` ↔ `:playing` ↔ `:paused`) and volume (`:unmuted` ↔ `:muted`).

```clojure
(def media-player
  (statechart {}
    (parallel {:id :player}
      (state {:id :playback}
        (state {:id :stopped}
          (transition {:event :play :target :playing}))
        (state {:id :playing}
          (transition {:event :pause :target :paused})
          (transition {:event :stop :target :stopped}))
        (state {:id :paused}
          (transition {:event :play :target :playing})
          (transition {:event :stop :target :stopped})))
      (state {:id :volume}
        (state {:id :unmuted}
          (transition {:event :toggle-mute :target :muted}))
        (state {:id :muted}
          (transition {:event :toggle-mute :target :unmuted}))))))
```

Visually identical to a `state` with two children — but `parallel` makes both children active. **Using `state` instead of `parallel` is the canonical ex04 footgun.**

---

## Mechanics in 30 seconds

- **`parallel` activates all children at once.** Each is a "region."
- **Configuration after `start!`:** `#{:player :playback :stopped :volume :unmuted}` — 5 elements (parallel + 2 regions + 2 active leaves). `:ROOT` excluded as usual.
- **Event broadcast:** every region's transitions are checked. Multiple regions can fire on a single event.
- **Region independence is structural.** `:volume` doesn't react to `:play` *because* nothing in `:volume` declares a `:play` transition. The library doesn't enforce independence — your declarations do.
- **Exit-the-whole-parallel rule:** a transition targeting outside the parallel exits ALL regions. Per SCXML's LCA computation.
- **Transitions on the `parallel` itself** (alongside regions, not inside them) fire whenever the parallel is active — a "machine-wide" handler.
- **`(t/in? env :parallel-id)` returns true** — the parallel's `:id` is in the configuration alongside its regions.

---

## Composes with

- **ex02's ancestor walk** applies independently per region for event matching.
- **ex05** adds multi-target transitions for precise re-entry into specific leaves across regions.
- **ex06 (history)** can be applied per region — each region can have its own history node.
- **ex08 (final / done.state)** for parallels: `done.state.<parallel-id>` fires only when ALL regions reach a final.

---

## Gotchas to remember

- **`state` instead of `parallel`** — chart compiles fine, but only one child is active. Test fails because the other region's state isn't in the config. The single most consequential element choice in a multi-region chart.
- **`:ROOT` is still excluded from the configuration** in parallel charts. Same rule as flat charts.
- **Cross-region transitions are weird.** A transition from one region's child to another region's child leaves the source state IN the configuration. SCXML's LCA semantics are subtle here — avoid the pattern.
- **First child = default initial.** If `:muted` is declared before `:unmuted` in `:volume`, the chart starts in `:muted`. Use `:initial` on the region to override.
- **Two regions handling the same event = both fire.** Sometimes desirable, often surprising.
- **Empty regions are legal but degenerate.** `(state {:id :region})` with no children adds `:region` to config but no atomic leaf for it.

Full list in [gotchas.md](./gotchas.md).

---

## Re-engage in 5 minutes

```clojure
(require '[exercises.ex04-parallel :as ex04])
(require '[com.fulcrologic.statecharts.testing :as t])

(def env (t/new-testing-env {:statechart ex04/media-player
                              :mocking-options {:run-unmocked? true}} {}))
(t/start! env)
(t/in? env :stopped)              ; → true   (playback region)
(t/in? env :unmuted)              ; → true   (volume region, simultaneous)
(t/in? env :player)               ; → true   (parallel is in config too)

(t/run-events! env :play)
(t/in? env :playing)              ; → true   (playback advanced)
(t/in? env :unmuted)              ; → true   (volume UNCHANGED — independence)

(t/run-events! env :toggle-mute)
(t/in? env :playing)              ; → true   (playback UNCHANGED)
(t/in? env :muted)                ; → true   (volume advanced)
```

If those six checks pass, ex04 is solid. Move on to [ex05 (multi-target transitions)](../ex05-multi-target/) for the `:target [:a :b]` form that lets you precisely specify leaves in multiple regions.
