# Gotchas — ex08: Final States, `done.state` Events, and Chart Exit

Silent failures, hidden behaviors, and easy-to-miss configurations introduced by ex08. See [`gotchas-overview.md`](../gotchas-overview.md) for the doc-type framing.

ex01–ex07 gotchas still apply. This file adds what `final` + `done.state` + chart-exit newly put on the table.

All entries verified empirically against `com.fulcrologic/statecharts` 1.2.25.

---

### 1. Finals can't have child elements — including transitions

**What you'll see.** You want a final state that can be "un-done" — exited via some event. You write:

```clojure
(final {:id :validated}
  (transition {:event :reset :target :validating}))
```

The chart fails to construct[^p6]:

```
Illegal children of :final with attributes {:id :validated}.
That node cannot have children of type(s): (:transition)
```

**What's really happening.** SCXML semantics treat finals as terminal: a final is *done*, and that's the end of its participation in the chart's execution. Allowing outgoing transitions from a final would violate that contract. The library enforces it at construction time.

**Defense.** If you want a state that "looks final" but allows escape, use a regular state with no outgoing transitions. It'll be stuck (no events move it forward), but it won't trigger `done.state.<parent>` and won't have the structural restrictions of `final`. Then have an ancestor's transition handle whichever event should re-route the chart.

Alternative: use a guard on the final's auto-event transition to conditionally suppress completion. But this gets tangled fast; usually the chart wants explicit completion, not conditional un-completion.

---

### 2. `:done.state.<id>` is just a regular event — and ex02's prefix matching can catch it

**What you'll see.** You're writing a chart with several `:done`-prefixed events (`:done.state.validation`, `:done.state.enrichment`, `:done.state.processing`). You declare a *catch-all* transition for cleanup:

```clojure
(transition {:event :done :target :cleanup})
```

You expect it to handle a custom `:done` event you fire externally. Instead, it catches *every* `:done.state.X` event the runtime synthesizes — because `:done` is a string-prefix of all of them. Your chart goes to `:cleanup` after the first region completes, not after your custom event.

**What's really happening.** ex02's string-prefix matching fallback (gotchas #1 in that exercise) treats `:done` as a prefix-matcher for any event whose name starts with the string `":done"`. The runtime's auto-fired `done.state.<id>` events all match.

**Defense.** Use the *full* event name in transitions that should catch auto-events: `:done.state.processing`, not `:done`. For custom events, avoid names that string-prefix with `:done.` or `:error.` — those are SCXML conventions for runtime-synthesized events. Picking `:cleanup` or `:user/done` instead of bare `:done` avoids the collision.

*See also:* [ex02 gotchas.md #1](../ex02-event-matching/gotchas.md), [ex06 gotchas.md #7](../ex06-history/gotchas.md) (same footgun in a different module).

---

### 3. Partial parallel completion fires only the *region's* done.state, not the parallel's

**What you'll see.** You have a parallel with two regions. One region completes; you expect `done.state.<parallel>` to fire (maybe as "first completion"). It doesn't fire until *both* regions are final.

**What's really happening.** The SCXML rule: `done.state.<parallel>` fires only when *all* regions of the parallel have reached a final state[^p5]. Per-region `done.state.<region>` events fire as each region completes, but the parallel's overall completion event waits for the last region.

```
:par with regions :r1 :r2
:r1 completes  → done.state.r1 fires (NOT done.state.par)
:r2 completes  → done.state.r2 fires AND done.state.par fires (now all regions final)
```

**Defense.** If you need "first completion" semantics (e.g., race conditions where the first to complete wins), don't use `done.state.<parallel>`. Listen for *each* per-region done event:

```clojure
(transition {:event :done.state.r1 :target :winner-is-r1})
(transition {:event :done.state.r2 :target :winner-is-r2})
```

If `:r1` finishes first, the first transition fires. The chart moves before `:r2` has a chance to complete. (Caveat: the first transition exits the parallel, which interrupts the still-running `:r2` region — its final never fires. That's the right semantics for "winner takes all.")

---

### 4. Top-level final exits the chart — *only* top-level

**What you'll see.** You declare a final inside a compound state, expecting it to exit the chart. The chart doesn't exit. Configuration retains the compound and other ancestors; `running?` stays `true`. You're confused.

**What's really happening.** Only finals that are *direct children of the chart root* trigger chart exit[^p4]. A final inside a compound fires `done.state.<parent>` (the auto-event) but leaves the chart running. The chart waits for either:

1. A transition matching the auto-event to move it elsewhere, OR
2. The chart's outer transitions to drive it to a top-level final.

**Defense.** Pick the right level for your final based on intent:

- **Final inside a compound** = "this sub-flow is done; the outer flow can react."
- **Final at chart root** = "the entire chart is done; terminate."

For the exercise: `:validated` and `:enriched` are *region-level* finals (signal "this region is done"). `:finished` is *top-level* (signal "chart is done"). Both are correct for their position.

---

### 5. The chart's `running?` flag distinguishes "exited" from "stuck"

**What you'll see.** Your test fires events and nothing happens. You're not sure if the chart exited or if it's just stuck at a leaf with no outgoing transitions.

**What's really happening.** Two distinct end-of-life states for a chart:

| Condition | Configuration | `running?` | Subsequent events |
| --- | --- | --- | --- |
| **Exited** (top-level final reached) | `#{}` (empty) | `false` | Silently dropped |
| **Stuck** (in a leaf state with no transitions) | `#{:state-id ...}` (non-empty) | `true` | Silently dropped unless an ancestor transition matches |

**Defense.** Inspect both:

```clojure
(defn running? [env]
  (-> env :env (get :com.fulcrologic.statecharts/working-memory-store)
      :storage deref :test (get :com.fulcrologic.statecharts/running?)))

(running? env)      ; false = chart exited; true = chart still alive
(t/in? env :foo)    ; check specific states even if "stuck"
```

For tests, `(running? env)` returns `false` is the clean "we reached chart end" signal. For "stuck without exiting," you'd see `(running? env) ⇒ true` plus a non-empty configuration that no longer responds to your events.

---

### 6. `done.state.<id>` fires *as an internal event* — it's not in the external queue

**What you'll see.** You add diagnostic logging to capture every external event fired against the chart. You expect to see `:done.state.processing` in your log. You don't — even though the chart reacts to it correctly.

**What's really happening.** Internal events (per `:type :internal`) are processed by the runtime's internal queue, separate from external events that `t/run-events!` puts on the external queue. The runtime synthesizes the auto-event and immediately dispatches it without going through the external pipeline.

**Defense.** If you want to log auto-events, add a transition that catches them (with a side-effecting `:cond` that always returns true):

```clojure
(transition {:event :done.state.processing
             :cond (fn [_ data] (println "auto-event:" (:_event data)) true)
             :target :complete})
```

The `:cond`'s side effect runs every time the transition is evaluated — which includes when the auto-event arrives. From your guard, you can inspect `(:_event data)` for the full event map.

---

### 7. After chart exit, `t/run-events!` doesn't error — just silently does nothing

**What you'll see.** Your chart reaches `:finished` (top-level final). You fire more events as part of test setup or teardown. The events appear to "succeed" (no exceptions) but nothing happens. You wonder if the chart is broken.

**What's really happening.** After chart exit, `running?` is `false` and the configuration is empty. `t/run-events!` accepts the call but the runtime has no active states to process the event against — it just drops it. No error, no warning, no log.

**Defense.** After verifying chart exit (`(running? env)` returns `false`), don't expect any further interaction. If your test sequence assumes more transitions after the chart exits, the test logic is wrong (or the chart's design is wrong — perhaps the final shouldn't be top-level).

---

### 8. `done-data` is mentioned in SCXML but rarely needed in this curriculum

**What you'll see.** Reading the SCXML spec or other library examples, you encounter `done-data` — a way to attach data to the auto-event. You wonder if you should be using it.

**What's really happening.** The library supports `done-data` (per the elements namespace). It lets a final declare data that becomes the `:data` of the auto-event. In the absence of `done-data`, the auto-event's `:data` is empty (`{}`)[^p7].

For the curriculum's exercise, `done-data` doesn't come up — the test only checks state, not auto-event data. For production charts, it's useful when:

- A parallel completing should pass aggregate results to the parent.
- A region's final should communicate which path it took (success/failure detail).

**Defense.** Don't add `done-data` unless you have a specific need to read it from the auto-event handler. Premature use clutters the chart definition.

---

> **Verified empirically (probes run during ex08 authoring):**
>
> - [^p4]: Top-level final → configuration empty + `running?` false
> - [^p5]: `done.state.<parallel>` fires only when ALL regions are final
> - [^p6]: Finals reject children; transitions inside throw at construction
> - [^p7]: Auto-event has `:data {}` by default (no `done-data`)
