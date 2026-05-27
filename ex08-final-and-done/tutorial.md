# Tutorial — ex08: Final States, `done.state` Events, and Chart Exit

The conceptual narrative behind exercise 8. After reading, open [`runbook.md`](./runbook.md) to build the chart and watch the tests go green.

ex08 is typical-density but introduces three new mechanics together: the `final` element, the auto-generated `done.state.<id>` events, and the chart-exit behavior when a top-level final is reached. The mechanics compose elegantly to express "task done" / "all tasks done" / "chart done."

---

## The story we're building

A processing pipeline. Two independent tasks run in parallel:

- **Validation** — validates incoming data.
- **Enrichment** — adds derived fields.

When *both* tasks finish, the pipeline moves to `:complete`. The user can then `:finalize` to fully terminate the chart.

The mechanics that make this work:

1. Each region's "done" state is a `final` element — when entered, it signals "this region is done" without needing an explicit event handler.
2. When *all* regions of a parallel reach `final`, the runtime auto-fires `done.state.<parallel-id>` — a synthesized internal event that's just a regular event for matching purposes.
3. A transition can listen for `done.state.processing` (the parallel's auto-event) and dispatch the chart to `:complete`.
4. A top-level `final` causes the chart to fully exit when entered — the configuration empties, `running?` becomes false.

What you'll have grounded by the end:

1. The **`final` element** — its declaration, its restriction (no children allowed).
2. The **`done.state.<id>` event** — when it fires, its shape, who handles it.
3. **Parallel completion semantics** — `done.state.<parallel-id>` fires only when *all* regions are final.
4. **Chart exit** — what happens when a top-level final is entered.
5. The composition: how these three mechanics enable the "wait for all tasks → react" pattern.

---

## Step 1 — `final` declares a terminal state

The element:

```clojure
(final {:id :validated})
```

Returns `{:id :validated, :node-type :final}`[^p1]. That's it — no body, no children. The library validates that finals don't have children: trying to declare a transition inside a final throws `Illegal children of :final ...`[^p6].

Finals can be declared anywhere a state can: as a child of a compound state, as a child of a region, or at the chart root. The library treats final entry slightly differently from regular state entry — it triggers a `done.state` event (Step 2 next).

Once the chart reaches a final, it can't transition *out* of that final via that final's own transitions — finals can't have transitions. The chart can only leave the final's region via the parent compound state's transitions (handling `done.state.<parent>`, for instance).

> **Common misconception** — "`final` is just a regular state with a flag set."
>
> It's a distinct element type with structural restrictions (no children, including no transitions). The runtime recognizes `:node-type :final` and applies special semantics: auto-fire `done.state.<parent>` on entry, exclude from history-recording in some cases, signal chart-exit at top level.

---

## Step 2 — Entering a final fires `done.state.<parent>`

When the chart enters a `final` element, the runtime synthesizes an **internal event** named `done.state.<parent-id>` and dispatches it[^p7]:

```clojure
;; Inside a compound :container with a final :done as a child
(state {:id :container}
  (state {:id :working} (transition {:event :go :target :done}))
  (final {:id :done})
  (transition {:event :done.state.container :target :next-state}))   ; handles the auto-event

(state {:id :next-state})
```

When `:go` fires and the chart enters `:done`:

1. `done.state.container` is automatically queued as an internal event.
2. The runtime processes the queue; the transition on `:container` listens for `:done.state.container` and fires.
3. Chart moves to `:next-state`[^p3].

Empirically, the event has this shape:

```clojure
{:type :internal
 :name :done.state.container
 :sendid :done                              ; the final's :id
 :data {}
 :com.fulcrologic.statecharts/event-name :done.state.container}
```

The `:sendid` is the final's own `:id`. The `:data` is empty unless the final declares a `done-data` element (covered as a footnote — not used in this exercise).

> **Common misconception** — "I need to manually raise the `done.state` event in on-entry."
>
> The runtime synthesizes it for you. You don't write `done.state.X` anywhere except in *transitions that match it*. Entry to the final is the trigger.

---

## Step 3 — Parallel completion: `done.state.<parallel>` fires only when ALL regions are final

In a parallel state, the auto-event semantics scale up: `done.state.<parallel-id>` fires only when **every** region has reached a final state[^p5].

For the exercise's chart:

```clojure
(parallel {:id :processing}
  (state {:id :validation}
    (state {:id :validating} (transition {:event :validation-ok :target :validated}))
    (final {:id :validated}))                       ; final inside :validation
  (state {:id :enrichment}
    (state {:id :enriching} (transition {:event :enrichment-ok :target :enriched}))
    (final {:id :enriched})))                       ; final inside :enrichment
```

Lifecycle:

1. `:validation-ok` fires → `:validation` reaches its final `:validated`. The runtime fires `done.state.validation`. But `:enrichment` is still in `:enriching` — not done. So `done.state.processing` does *not* fire yet.
2. `:enrichment-ok` fires → `:enrichment` reaches its final `:enriched`. Now BOTH regions are final. The runtime fires `done.state.enrichment` *and* `done.state.processing`.
3. A transition listening for `:done.state.processing` (declared on `:processing`'s parent, `:pipeline`) catches the event and dispatches.

This is the "wait for all parallel work to complete" pattern. The runtime computes "all regions final" automatically; you don't need to track completion in the data model or check after every event.

> **Common misconception** — "`done.state.<parallel>` fires when any one region completes."
>
> No — only when *all* regions are final[^p5]. Partial completion fires the per-region `done.state.<region>` event but not the parallel's overall one.

---

## Step 4 — Top-level `final` exits the chart

When the chart reaches a final that's a **direct child of the chart root** (not inside any compound), the chart itself terminates:

```clojure
(statechart {}
  (state {:id :start} (transition {:event :go :target :end}))
  (final {:id :end}))            ; top-level final
```

After `:go`:

- The configuration becomes empty: `#{}`[^p4].
- `running?` in the working memory becomes `false`.
- Subsequent events to the chart are silently dropped (the chart is over).

The exercise's chart uses this pattern: `:finished` is a top-level final, so `:finalize` transitioning to `:finished` terminates the chart entirely.

> **Common misconception** — "Reaching any final exits the chart."
>
> Only a top-level final terminates the chart. A final inside a compound or region exits *that container* (triggering its `done.state` event) but doesn't stop the chart from running. The chart keeps processing events until a top-level final is entered.

---

## Step 5 — Composing: the full pipeline

The exercise's chart layers all three mechanics:

```
:pipeline (compound)
  ├─ :processing (parallel)
  │    ├─ :validation
  │    │    ├─ :validating → :validated on :validation-ok
  │    │    └─ (final) :validated
  │    └─ :enrichment
  │         ├─ :enriching → :enriched on :enrichment-ok
  │         └─ (final) :enriched
  ├─ :complete
  │    └─ on :finalize → :finished
  └─ on :done.state.processing → :complete   (handles auto-event)
(final) :finished    ← top-level — exits chart
```

The flow:

1. Both regions of `:processing` start in their working states (`:validating`, `:enriching`).
2. Each region's `*-ok` event drives it to its `final`.
3. When *both* regions are final, `:done.state.processing` fires; the transition on `:pipeline` catches it; chart moves to `:complete`.
4. From `:complete`, `:finalize` targets `:finished` (top-level final).
5. Chart exits.

The architectural beauty: no explicit "completion tracking" in the data model, no manual coordination between regions, no event broadcasting yet for completion. The runtime composes the per-region finalization into a single broadcast event for the parallel as a whole.

The chart, as Clojure (the solution):

```clojure
(def processing-pipeline
  (statechart {}
    (state {:id :pipeline}
      (parallel {:id :processing}
        (state {:id :validation}
          (state {:id :validating}
            (transition {:event :validation-ok :target :validated}))
          (final {:id :validated}))
        (state {:id :enrichment}
          (state {:id :enriching}
            (transition {:event :enrichment-ok :target :enriched}))
          (final {:id :enriched})))
      (state {:id :complete}
        (transition {:event :finalize :target :finished}))
      (transition {:event :done.state.processing :target :complete}))
    (final {:id :finished})))
```

---

## Step 6 — The `done.state` event name and prefix-matching surprise

ex02 surfaced the **string-prefix matching fallback** in `name-match?`. ex08's auto-fired events have a real chance of biting via this fallback:

```clojure
(transition {:event :done :target :somewhere})
```

This transition listens for `:done`. But via the string-prefix fallback, `:done` matches *any* event whose name starts with the string `":done"` — including `:done.state.processing`, `:done.state.validation`, `:done.invoke.<id>` (ex09 territory). A learner who writes a transition listening for `:done` thinking it'll only fire on a custom `:done` event will accidentally catch every auto-generated done event.

**Defense:** when writing transitions that should match auto-events, use the *full* event name (`:done.state.processing`), not a prefix. When writing transitions for custom events, avoid names that string-prefix with `:done.` or `:error.` (which are SCXML conventions for auto-events).

The exercise's chart uses `:done.state.processing` explicitly — no surprises. But it's worth knowing the cross-module collision before you write your own done-listening transitions.

> **Common misconception** — "`:done` is a reserved keyword."
>
> It isn't reserved — but it's an awkward name to choose for a transition's `:event` because `:done.state.X` and `:done.invoke.X` events from the runtime will all match it via the prefix fallback. The library wouldn't warn you.

*See also:* [ex02 gotchas.md #1](../ex02-event-matching/gotchas.md) (the original string-prefix-fallback gotcha).

---

## What you should be able to explain

When you finish the runbook and the tests pass, you should be able to explain — out loud, in plain English — each of these:

- **What `final` declares and what it forbids.** A terminal state; cannot have children (including transitions).
- **When `done.state.<parent>` fires.** When entering a final inside that parent — synthesized by the runtime as an internal event.
- **The parallel completion rule.** `done.state.<parallel>` fires only when *all* regions reach a final.
- **What "chart exit" means.** Entering a top-level final makes the configuration empty and `running?` false; subsequent events are dropped.
- **Why the chart uses two layers of final (per-region + top-level).** Per-region finals signal "this task done"; top-level final signals "chart done." Different semantics, both useful.

If any of these still feel hazy after the runbook, [`glossary.md`](./glossary.md) has the entries to revisit. Then [`quiz.md`](./quiz.md) checks recall, and [`gotchas.md`](./gotchas.md) collects the silent footguns.

---

> **Verified empirically (probes run during ex08 authoring):**
>
> - [^p1]: `(final {:id :x})` returns `{:id :x, :node-type :final}`
> - [^p3]: Entering a final inside a compound state fires `done.state.<parent>`, which transitions on the parent can match
> - [^p4]: Reaching a top-level final empties the configuration and sets `running?` to false
> - [^p5]: `done.state.<parallel>` fires only when *all* regions of the parallel are final — partial completion fires per-region events but not the parallel's
> - [^p6]: Finals can't have children — declaring a transition inside throws `Illegal children of :final ...`
> - [^p7]: The auto-event has `:type :internal :name :done.state.<parent-id> :sendid <final-id> :data {}`
