# Glossary — ex07: Internal vs External Transitions

Terms introduced (or first deeply covered) in ex07. Cross-cutting vocabulary lives in the top-level [`glossary.md`](../glossary.md); ex01–ex06 covered states, transitions, configurations, on-entry/on-exit (via ex03b/ex07 script use). This file adds what ex07 newly puts on the table.

Each entry has a 0–9 criticality rating (see the top-level glossary for the scale).

---

### `:type :external` (default)  *(criticality: 8)*

A transition's `:type` value. When omitted, every transition defaults to `:external`[^p3].

Behavior: when the transition fires, the runtime computes an **exit set** that includes the transition's source state *and* its current descendant chain. The source state exits, the target's path is entered. If the source state has on-exit / on-entry actions, they fire on every transition.

For most chart designs, external is correct and matches intuition. The cases where external is *wrong* are precisely those ex07 introduces — transitions declared on a compound state whose on-entry/on-exit shouldn't re-run on every within-compound move.

### `:type :internal`  *(criticality: 8)*

A transition's `:type` value (opt-in; not the default). When the source state is a compound state AND the target is a strict descendant, internal transitions **keep the source state in the configuration** — its on-exit and on-entry do not fire when the transition fires. Only the active descendant changes[^p1].

```clojure
(state {:id :editor}
  (on-entry {} (script {:expr load-doc!}))
  (transition {:event :edit :target :editing :type :internal})
  (state {:id :viewing})
  (state {:id :editing}))
```

When `:edit` fires while in `:viewing`, the chart moves to `:editing` without re-firing `load-doc!`. The compound `:editor` stays "in."

### narrow applicability of `:type :internal`  *(criticality: 7)*

Internal is meaningful only in one combination of structural elements:

| Source | Target | Internal does anything? |
| --- | --- | --- |
| Compound state | Strict descendant | **Yes** — source stays in configuration[^p1] |
| Compound state | Non-descendant (sibling, distant ancestor, top-level state) | **No** — silently ignored; behaves as external[^p4] |
| Atomic state | Anything | **No** — atomic states have no descendants; internal/external are equivalent |
| Any state | Itself (self-target) | **No** — internal self-target still exits and re-enters[^p6] |

In all "No" cases, the library accepts `:type :internal` without complaint and silently produces the same behavior as `:type :external`. The annotation is a no-op.

### exit set  *(criticality: 6)*

The set of currently-active states that a transition will exit when it fires. Computed by the runtime from the transition's source, target, and the chart's structure.

For an **external transition** declared on state `S` targeting `T`:
- LCA = the lowest common ancestor of `S` and `T` (or `S`'s *parent* if `T` is a descendant of `S`).
- Exit set = currently-active descendants of LCA, including `S` itself.

For an **internal transition** declared on state `S` targeting `T` (where `T` is a descendant of `S`):
- LCA = `S` itself.
- Exit set = currently-active descendants of `S`, EXCLUDING `S` itself.

This is the SCXML-spec rule that explains all of ex07's behavior. The exit set determines which states' on-exit actions fire; the entry set (computed similarly for the target's path) determines which on-entry actions fire.

You don't need to compute exit sets manually for the exercise. The high-level rule is sufficient: **internal keeps the compound source active**; external exits it.

### LCA (Least Common Ancestor)  *(criticality: 5)*

The deepest state in the chart that's an ancestor of both the transition's source and target. SCXML's transition algorithm uses the LCA to compute exit and entry sets.

For external transitions: LCA = parent of source (or shared ancestor for cross-region cases). The source state is in the exit set.

For internal transitions where source is compound and target is descendant: LCA = source itself. The source is NOT in the exit set.

This is the most "mathy" piece of SCXML semantics; most chart authors never need to compute LCAs explicitly. Knowing the concept exists is enough — when "why does my compound state re-enter?" comes up, "LCA computation" is the conceptual answer.

### where the transition is declared (vs. its source)  *(criticality: 6)*

A transition's "source state" is the *state it's declared inside*, not the state the chart is in when it fires. This matters for `:type :internal`:

- Transition declared on `:editor` (compound, with children `:viewing` and `:editing`): source = `:editor`. Targeting `:editing` is targeting a descendant → internal is meaningful.
- Transition declared on `:viewing` (child of `:editor`): source = `:viewing`. Targeting `:editing` is targeting a sibling → internal vs external doesn't matter[^p5].

Where you put the transition determines what `:type :internal` controls. The exercise puts switching transitions on the parent for a reason — that's where internal does its job.

### on-entry / on-exit (re-fire semantics)  *(criticality: 7)*

ex03b introduced `on-entry`. ex07 makes the timing matter:

- **on-entry fires every time the state is entered.** First entry on `start!`, every subsequent re-entry via a transition.
- **on-exit fires every time the state is exited.** Including external self-transitions that exit and re-enter the same state.

If your on-entry/on-exit has expensive side effects (DB call, API request, animation start), you care about *exactly* when they fire — and `:type :internal` is the mechanism that lets you transition without triggering them.

### the editor pattern  *(criticality: 4)*

A common chart shape ex07 uses as its subject:

> A compound state with one-time entry side effects (loading), one-time exit side effects (saving), and child states for sub-modes. Switches between sub-modes should be internal (no re-load, no premature save).

You'll see this shape in:
- Document editors (load on open, save on close)
- Modal dialogs (allocate resources on open, release on close)
- Pages with tabs (fetch data once, switch tabs internally)
- Multi-step forms (initialize once, navigate between steps internally)

`:type :internal` on the within-compound transitions is the canonical fix. The pattern shows up frequently enough in real charts that it's worth keeping in mind even outside this exercise.

---

> **Verified empirically (probes run during ex07 authoring):**
>
> - [^p1]: `:type :internal` with source=compound, target=descendant — compound stays active; on-entry/on-exit don't re-fire
> - [^p3]: When `:type` is omitted, library defaults to `:external` (verified by counter equivalence)
> - [^p4]: `:type :internal` with target outside source's descendant chain is silently ignored
> - [^p5]: Transition declared on a child state doesn't trigger the parent's re-entry regardless of `:type`
> - [^p6]: `:type :internal` self-target still exits and re-enters
