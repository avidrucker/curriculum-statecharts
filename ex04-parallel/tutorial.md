# Tutorial — ex04: Parallel Regions

The conceptual narrative behind exercise 4. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex04 is the first **dense** module — the chart has more structure than anything you've seen so far, and the concept of "the chart is in multiple atomic states at once" reshapes how you reason about every later exercise. Take it slowly.

---

## The story we're building

A media player with two **independent** concerns:

1. **Playback**: `:stopped` ↔ `:playing` ↔ `:paused`. The user can play, pause, stop, resume.
2. **Volume**: `:unmuted` ↔ `:muted`. The user can mute/unmute at any time.

The key word is **independent**. Muting doesn't affect playback — you can mute while paused, mute while playing, unmute while stopped. Pausing doesn't affect mute — you can pause without un-muting first.

Modeling this with the chart vocabulary you've used so far (`state`, `transition`, `compound state`) would be painful. You'd need a state for every combination: `:playing-muted`, `:playing-unmuted`, `:paused-muted`, `:paused-unmuted`, `:stopped-muted`, `:stopped-unmuted`. Six states for two independent concerns; a third concern (say, `:fullscreen` ↔ `:windowed`) doubles it to twelve. The combinatorial explosion is the canonical reason finite state machines are insufficient for non-trivial UI.

**The parallel state element** is the statechart solution. It lets you declare "these regions are all active simultaneously, each evolving on its own." The chart becomes the *product* of independent regions, not their permutation.

What you'll have grounded by the end:

1. The **`parallel`** element — its grammar and semantics.
2. The **region** concept — each child of a `parallel` is an independent compound state.
3. The **multi-element configuration** — the chart is in *every* active atomic state across all regions, plus their ancestors, simultaneously.
4. **Event broadcast** — every region with a matching transition processes the event independently.
5. **Region independence** — what makes parallel different from a compound state with siblings.
6. The **exit-the-whole-parallel rule** — a transition out of a region (targeting outside the parallel) collapses *every* region at once.

---

## Step 1 — `parallel` looks like `state` but isn't

Syntactically, `parallel` is declared just like a compound state:

```clojure
(parallel {:id :player}
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

Looks identical to:

```clojure
(state {:id :player}
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

But the runtime treats them very differently:

| | `state` (compound) | `parallel` |
| --- | --- | --- |
| How many children active at once? | **Exactly one** | **All of them** |
| When entered, what happens? | Enter the initial child (per `:initial` or document order) | Enter **every** child simultaneously |
| When an event arrives? | The runtime walks the active child's transitions, then ancestors | The runtime visits **every** region's active state and broadcasts the event there |

The visual identical-ness is a deliberate library choice — but pedagogically it's a trap. **Choosing `state` when you meant `parallel` is the canonical ex04 footgun** (see [`gotchas.md` #1](./gotchas.md)).

> **Common misconception** — "Parallel and compound states are interchangeable; one just lets you have multiple children."
>
> Compound states with multiple children still only have *one active child at a time*. Adding a second child to a compound state doesn't make both active — it just adds an alternative initial. `parallel` is genuinely different machinery.

---

## Step 2 — Each child of a `parallel` is a "region"

The terminology: the immediate children of a `parallel` element are called **regions**. In the media-player chart:

- `:playback` is a region.
- `:volume` is a region.

Regions are themselves compound states. They have their own children (`:stopped`, `:playing`, `:paused` for `:playback`; `:unmuted`, `:muted` for `:volume`), their own initial-state-by-document-order rule, their own transitions.

What makes them **regions** rather than just children: when the `:player` parallel is entered, the runtime enters **every** region at once, each landing in its own initial state.

```
Enter :player (parallel)
  ↓
Enter :playback (region #1) — and its initial child :stopped
AND
Enter :volume (region #2) — and its initial child :unmuted
```

Both regions are active. The configuration immediately after `(t/start! env)` is:

```
#{:player :playback :stopped :volume :unmuted}
```

Five elements. Two atomic leaves (`:stopped` and `:unmuted`), each region (the compounds), and the parallel itself.

`:ROOT` is excluded, same rule as ex01.

> **Common misconception** — "A region's children are regions too."
>
> They're not. **Only the *immediate* children of a `parallel` are regions.** A region's own children (the playback states `:stopped`, `:playing`, `:paused`) are just regular compound-state children — one active at a time within their region.

---

## Step 3 — `t/in?` against multiple active states

ex02 established that `t/in?` is a simple `(contains? configuration state-name)`. ex04 finally exercises this on a configuration with multiple atomic leaves:

```clojure
(t/in? env :stopped)   ; → true   (:stopped is in the configuration)
(t/in? env :unmuted)   ; → true   (:unmuted is also in the configuration)
(t/in? env :player)    ; → true   (the parallel is too)
(t/in? env :playback)  ; → true   (the region is too)
(t/in? env :muted)     ; → false  (:muted is not active in :volume)
```

Both `:stopped` and `:unmuted` returning true is **not** because `t/in?` got smarter. The configuration set genuinely contains both. The set has five elements at this moment.

The exercise's first assertion exploits this:

```clojure
(testing "starts in :stopped and :unmuted (two active leaves)"
  (is (t/in? env :stopped))
  (is (t/in? env :unmuted)))
```

Both assertions pass *simultaneously* because both are truthy. In a flat (non-parallel) chart, exactly one atomic-state assertion can be true at a time. In a parallel chart, *each region contributes exactly one* atomic state to the set.

> **Common misconception** — "`(t/in? env :stopped)` and `(t/in? env :unmuted)` being both true means the chart is in some weird hybrid state."
>
> It means the configuration set has *both* IDs. The chart is in `:stopped` AND `:unmuted` simultaneously — one in each region. That's the entire point of parallel: orthogonal state, independently tracked.

---

## Step 4 — Event broadcast: every region with a matching transition fires

When you send `:play` to a chart with two regions, the runtime visits **every** region and asks "do you have a matching transition?" — and any region that does fires its transition.

For the media player:

```
Configuration:  #{:player :playback :stopped :volume :unmuted}
Event:          :play
```

- `:playback` region: active leaf is `:stopped`. `:stopped` has a transition on `:play` targeting `:playing`. **Fires.** `:stopped` exits, `:playing` enters.
- `:volume` region: active leaf is `:unmuted`. `:unmuted` has no transition on `:play`. **No match.**

Result:

```
New configuration:  #{:player :playback :playing :volume :unmuted}
```

`:playback` updated (`:stopped` → `:playing`); `:volume` untouched. That's what "independence" means in practice — each region's reaction to an event is its own concern.

### What if *both* regions match?

Both regions fire independently. The library will send `:toggle-mute` to both regions; if both have a matching transition, both will fire. For the media player, only `:volume` has a `:toggle-mute` transition, so it's a single-region effect. But for charts where *two* regions both care about the same event, you get **simultaneous parallel transitions** — both regions update.

This becomes a powerful pattern (one event, multiple coordinated updates) and a footgun (one event accidentally affecting two regions that should be independent). See [gotchas.md #4](./gotchas.md).

> **Common misconception** — "The chart processes events sequentially — one region at a time, then the next."
>
> Conceptually, every region with a matching transition processes the event in the same processing cycle. From the test's view, both updates appear in the post-event configuration simultaneously.

---

## Step 5 — Independence is structural, not enforced

The reason `:toggle-mute` doesn't affect playback is *purely* that the `:playback` region's states don't declare a `:toggle-mute` transition. The chart doesn't have a "these regions are independent" declaration — independence is the natural result of regions only transitioning on events they explicitly handle.

If you accidentally declare a `:toggle-mute` transition inside one of `:playback`'s states (e.g., `:playing` has `:toggle-mute → :stopped`), then *yes*, muting would now also stop playback. The library wouldn't warn you — your chart would just have surprising behavior.

**Defense:** when adding a new event to a parallel chart, consciously decide which regions should respond to it. If only one should, only declare the transition there. If multiple should, declare it in each (and accept that they'll all fire on each event arrival).

---

## Step 6 — The exit-the-whole-parallel rule

The most surprising rule in parallel charts: **a transition whose `:target` is outside the parallel element exits *all* regions, not just the one the transition was declared in.**

```clojure
(parallel {:id :player}
  (state {:id :playback}
    (state {:id :stopped}
      (transition {:event :shutdown :target :off})))   ; targets :off (outside the parallel)
    (state {:id :playing} ...)
    (state {:id :paused} ...))
  (state {:id :volume}
    (state {:id :unmuted} ...)
    (state {:id :muted} ...)))

(state {:id :off})    ; outside :player
```

When `:shutdown` arrives while the chart is in `:stopped` (in `:playback`):

- `:stopped` exits.
- The entire `:playback` region exits.
- The entire `:volume` region exits (taking whatever active leaf it had with it).
- The `:player` parallel itself exits.
- `:off` is entered.

After this, the configuration is just `#{:off}` — the parallel is gone.

This rule is the SCXML-standard behavior (driven by the "LCA" — Least Common Ancestor — algorithm for transition exit sets). The intuition: a transition targets a state. To enter that state, the chart must exit *everything* outside the target's ancestor chain. For a target outside the parallel, that means exiting the parallel — which exits both regions.

> **Common misconception** — "A transition out of one region should leave the other region intact."
>
> It doesn't. Once a transition targets outside the parallel, the *whole* parallel exits. To preserve a region while changing another, the transition's `:target` must stay *inside* the parallel.

This is a load-bearing rule. ex06 (history states) will give you tools to *re-enter* a parallel later while preserving each region's last-active state. For ex04, just know the rule.

---

## Step 7 — Transitions on the parallel itself

You can declare transitions *on* the `parallel` element (not on a region or a region's child):

```clojure
(parallel {:id :player}
  (transition {:event :hard-reset :target :off})   ; transition on the parallel itself
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

When `:hard-reset` arrives in *any* configuration containing `:player`, this transition fires and (per Step 6) collapses the whole parallel. The exercise's chart doesn't use this pattern, but it's a real idiom — "events that affect the whole multi-region machine, regardless of which region's leaf is currently active."

Verification: empirically (probe P7), declaring `(transition {:event :exit-all :target :done})` as a direct child of a `parallel` and then firing `:exit-all` collapses every region and lands at `:done`.

---

## Step 8 — Putting it together

The media-player chart, in plain English:

> A media player has two independent dimensions: playback (stopped, playing, paused) and volume (unmuted, muted). They evolve on their own — pausing doesn't unmute; muting doesn't stop playback. Both start in their initial state when the player is turned on.

The chart, as a tree:

```
:player (parallel)
  ├─ :playback (region)
  │    ├─ :stopped → :playing on :play
  │    ├─ :playing → :paused on :pause, → :stopped on :stop
  │    └─ :paused  → :playing on :play,  → :stopped on :stop
  └─ :volume (region)
       ├─ :unmuted → :muted   on :toggle-mute
       └─ :muted   → :unmuted on :toggle-mute
```

The chart, in Clojure (the solution):

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

The runbook walks you through building this incrementally.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **Why `parallel` is not just "compound state with multiple active children."** Compound states have one active child at a time; parallels have all of them. The structural decision (which element to use) determines the runtime semantics.
- **What's in the configuration set after `(t/start! env)`.** The parallel's ID, every region's ID, every region's active atomic leaf — five elements for this chart.
- **Why `:play` doesn't affect `:unmuted`.** The `:volume` region has no transition on `:play`. Independence is structural.
- **What happens if a transition targets a state outside the parallel.** The whole parallel exits — both regions, the parallel itself. SCXML's LCA rule.
- **What it would take to declare a chart-wide event.** A transition on the `parallel` element itself.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns specific to parallel.
