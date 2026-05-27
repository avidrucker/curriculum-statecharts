# Tutorial — ex05: Multi-Target Transitions

The conceptual narrative behind exercise 5. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex05 is a "typical" density module — the chart is small (an extension of ex04's media player), but the mechanic is subtle. The honest punchline: **for this exercise's test, multi-target isn't strictly necessary** — but knowing *why* the test passes anyway is the load-bearing lesson.

---

## The story we're building

The ex04 media player, with one addition: a `:reset` event that returns the player to its initial state — `:stopped` (in `:playback`) *and* `:unmuted` (in `:volume`) — regardless of where it was before.

The straightforward expression is a transition whose `:target` is **a vector of state IDs**:

```clojure
(transition {:event :reset :target [:stopped :unmuted]})
```

The transition declares itself on `:player` (the parallel itself, not inside a region). When `:reset` arrives, the chart re-enters `:player`, with `:stopped` chosen for `:playback` and `:unmuted` chosen for `:volume`.

What you'll have grounded by the end:

1. **The vector form of `:target`** — what it does, where it's needed.
2. **The relationship between multi-target and default-initial entry** — multi-target *overrides* defaults; without it, regions take their default initials.
3. **The case where multi-target is genuinely needed** vs. the case where single-target works because the defaults align.
4. **One serious footgun** — multi-target with two targets in the same region produces an invalid configuration that the library doesn't catch.

---

## Step 1 — `:target` accepts a vector

ex01–ex04 used `:target` as a single keyword:

```clojure
(transition {:event :play :target :playing})
```

ex05 introduces the vector form:

```clojure
(transition {:event :reset :target [:stopped :unmuted]})
```

Semantically: when this transition fires, the chart enters *both* `:stopped` AND `:unmuted` (which are in different regions of the same parallel). Both end up in the active configuration simultaneously.

The library accepts either form everywhere `:target` is used. Internally, a single-keyword `:target` is normalized to a vector of one element — `(if (keyword? target) [target] target)` per the `transition` element's source. So `:target :stopped` and `:target [:stopped]` are equivalent.

> **Common misconception** — "Multi-target is a special transition flavor with different semantics."
>
> It isn't. The `transition` element is the same as before; the `:target` value just supports vectors. Internally the library always works with a vector. The single-keyword form is sugar for "exactly one target."

---

## Step 2 — Why you'd reach for multi-target

The exit-the-whole-parallel rule (ex04 Step 6) says: a transition whose target is outside a parallel exits the whole parallel and re-enters at the target. When the target is *inside* the parallel — say, the `:reset` transition on `:player` targeting `:stopped` (a state in `:playback`) — the chart exits the parallel, re-enters `:player`, and the runtime must pick an active leaf for *every* region.

Without an explicit target for `:volume`, the runtime falls back to `:volume`'s **default initial** (first child in document order, or whatever `:initial` declares). For the ex04 chart, that's `:unmuted`.

For ex05's test, that's enough: `:reset` should land in `:stopped` and `:unmuted`. Single-target on `:stopped` would re-enter `:volume` and default to `:unmuted`. **The test would pass.**

Multi-target is for the case where the default isn't what you want. Imagine the chart declared `:volume` differently:

```clojure
(state {:id :volume}
  (state {:id :muted}            ; ← now :muted is the document-order first
    (transition {:event :toggle-mute :target :unmuted}))
  (state {:id :unmuted}
    (transition {:event :toggle-mute :target :muted})))
```

Now `:volume`'s default initial is `:muted`. A single-target `:reset :target :stopped` would land the chart in `:stopped` *and* `:muted` — not what you wanted. To force `:unmuted`, you need:

```clojure
(transition {:event :reset :target [:stopped :unmuted]})
```

That's multi-target's purpose: **precise control over the active leaf in every region**, independent of each region's default initial.

> **Common misconception** — "I need multi-target because the chart has multiple regions."
>
> You don't always. Multi-target is needed when you want non-default initials in some region. If the defaults happen to align with what you want, single-target is enough. The exercise's test would pass with single-target *because* `:unmuted` is the default. Knowing this prevents reaching for multi-target by reflex.

---

## Step 3 — Where the `:reset` transition lives

The `:reset` transition is declared **on the `:player` parallel itself**, as a direct child:

```clojure
(parallel {:id :player}
  (transition {:event :reset :target [:stopped :unmuted]})       ; ← here
  (state {:id :playback} ...)
  (state {:id :volume} ...))
```

ex04 introduced this pattern in Step 7: transitions on the `parallel` element listen for events regardless of which atomic leaves each region currently holds. `:reset` should fire whether the chart is in `:playing :muted`, `:paused :unmuted`, or any other state — so it goes on the parallel.

If the transition were instead inside one region's child (say, `:playing`), it would only fire when the chart was in `:playing`. The parallel-level placement makes it work from any state inside `:player`.

> **Common misconception** — "I should put `:reset` inside each region (broadcast)."
>
> You don't need to. Putting one transition on the parallel covers every region in one declaration. Event broadcast (ex04) would also work — declaring `:reset` in each region — but it's more declarative noise for the same outcome.

---

## Step 4 — Order in the target vector doesn't matter

The two forms

```clojure
(transition {:event :reset :target [:stopped :unmuted]})
(transition {:event :reset :target [:unmuted :stopped]})
```

are semantically identical (empirically verified). The library doesn't process targets in left-to-right order; it figures out which region each target belongs to and activates them all in the same processing cycle.

For readability, the convention is to list targets in the same order as the regions appear in the chart definition (here: `:playback` first, so `:stopped` first; `:volume` second, so `:unmuted` second). This is a *style* choice with no semantic weight.

---

## Step 5 — The same-region footgun

The library doesn't validate that your multi-target vector hits *different* regions. You can write:

```clojure
(transition {:event :reset :target [:stopped :playing]})
```

…where both `:stopped` and `:playing` are atomic states in `:playback` (the same region). The chart loads. The transition fires. **The configuration ends up with both `:stopped` AND `:playing` active simultaneously inside `:playback`.**

This is structurally invalid — a compound state should have exactly one active child at a time — but the library doesn't catch the inconsistency. The chart enters a logically-broken state where both `(t/in? env :stopped)` and `(t/in? env :playing)` return true.

The chart typically *self-corrects* on the next event whose transition exits one of the two active states (e.g., firing `:play` from `:stopped` exits `:stopped`, leaving the chart legally in just `:playing`). But during the broken window, anything querying the configuration sees both states active — assertions can pass for the wrong reason.

**Defense.** In a multi-target `:target` vector, every target must be in a **different region** of the parallel. The library trusts you to enforce this. See [`gotchas.md` #2](./gotchas.md) for the empirical demonstration.

> **Common misconception** — "The library will reject same-region multi-target."
>
> It won't. The validator catches `:target` keywords that don't reference any state at all — but it doesn't check region uniqueness. Same-region multi-target compiles and runs; it just produces a broken configuration.

---

## Step 6 — Putting it together

The chart, in plain English:

> Same as ex04's media player, plus: a `:reset` event that returns the chart to its initial state (`:stopped` and `:unmuted`), no matter what it was doing.

The chart, as a tree (the new piece highlighted):

```
:player (parallel)
  ├─ on :reset → [:stopped :unmuted]       ← NEW
  ├─ :playback (region — unchanged from ex04)
  │    ├─ :stopped → :playing on :play
  │    ├─ :playing → :paused on :pause, → :stopped on :stop
  │    └─ :paused  → :playing on :play,  → :stopped on :stop
  └─ :volume (region — unchanged from ex04)
       ├─ :unmuted → :muted   on :toggle-mute
       └─ :muted   → :unmuted on :toggle-mute
```

The chart in Clojure (the solution):

```clojure
(def media-player-with-reset
  (statechart {}
    (parallel {:id :player}
      (transition {:event :reset :target [:stopped :unmuted]})
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

The runbook walks you through copying ex04's chart and adding the `:reset` transition.

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **What multi-target's `:target` value looks like.** A vector of state IDs. The library normalizes a single keyword to a one-element vector internally.
- **Why this specific exercise could be solved with `:target :stopped` (single).** `:unmuted` is the default-initial of `:volume`, so re-entering the parallel with only an explicit target for `:playback` lands `:volume` in `:unmuted` as a fallback.
- **When you'd genuinely need multi-target.** When the default initial in some region isn't what you want — e.g., to land in `:muted` instead of `:unmuted` while resetting `:playback`.
- **Why the `:reset` transition goes on the `parallel` element, not inside a region.** So it fires regardless of which atomic leaves each region is currently in.
- **Why `:target [:stopped :playing]` is a footgun.** Both targets are in the same region; the library doesn't validate this; the chart ends up with two atomic states simultaneously active in `:playback`, which is structurally invalid.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns.
